# nanoGPT 架构与核心模块交互总结

> 仓库：`karpathy/nanoGPT` — 一个用于复现 / 微调 GPT-2 的极简 PyTorch 实现。整个工程只有 **一个模型文件 `model.py` (~300 行)** 和 **一个训练脚本 `train.py` (~330 行)**，强调"读得完、改得动、能跑通"。

---

## 1. 项目原理

nanoGPT 是 **decoder-only Transformer 语言模型** 的最小可用复现，遵循 GPT-2 的设计：

| 维度 | 设计 |
|------|------|
| 任务目标 | 自回归语言建模：给定前 $t$ 个 token，预测第 $t{+}1$ 个 token 的分布。损失为 token 级 cross-entropy。 |
| 结构范式 | Decoder-only Transformer，Pre-LayerNorm（先 LN 再 Attn/MLP），残差相加。 |
| 注意力 | Causal multi-head self-attention；PyTorch ≥ 2.0 时调用 `scaled_dot_product_attention`（FlashAttention 路径），否则手写 softmax + 下三角 mask。 |
| 位置信息 | 学习式绝对位置嵌入 `wpe`（不是 RoPE / ALiBi）。 |
| 权重绑定 | `lm_head.weight` 与 token embedding `wte.weight` 共享（weight tying）。 |
| 优化器 | AdamW，二维参数（权重矩阵 + embedding）做 weight decay，bias / LayerNorm 不做；CUDA 上启用 fused 实现。 |
| 学习率调度 | 线性 warmup → cosine decay 到 `min_lr`。 |
| 训练精度 | `torch.amp.autocast`（bfloat16 优先，float16 配 `GradScaler`），并启用 TF32。 |
| 加速 | `torch.compile`（PyTorch 2.0 图编译）+ FlashAttention + 可选 DDP 多卡 / 多机训练。 |
| 数据 | 预处理阶段把语料 BPE 化（`tiktoken`）或字符化，序列化为 `uint16` 的扁平 `train.bin` / `val.bin`，训练时用 `np.memmap` 随机切片，零拷贝喂入 GPU。 |
| 训练 vs 推理 | 训练时计算全序列 logits 与 loss；推理时仅对最后一个时间步做 `lm_head`，再做 `top-k` + `temperature` 采样自回归生成。 |

设计哲学：**配置即代码**（`configurator.py` 用 `exec` 把 CLI / 配置文件直接覆盖到 `globals()`），**模型即一个 `nn.Module`**（没有 Trainer 抽象，没有 lightning，没有 hydra）。

---

## 2. 仓库文件结构

```
nanoGPT/
├── model.py                # GPT / Block / Attention / MLP / LayerNorm，单文件模型
├── train.py                # 训练入口：数据加载 + 优化器 + DDP + AMP + 调度 + 检查点
├── sample.py               # 加载 checkpoint 或 HF GPT-2，自回归采样
├── bench.py                # 不保存 checkpoint 的最小训练循环，用于性能基准
├── configurator.py         # CLI / py 文件覆盖 globals() 的"穷人版"配置系统
├── config/                 # 各任务预设：train_gpt2, train_shakespeare_char, finetune_shakespeare, eval_gpt2*
└── data/
    ├── openwebtext/prepare.py        # HF datasets + tiktoken BPE → train.bin / val.bin
    ├── shakespeare/prepare.py        # 小语料 BPE 化
    └── shakespeare_char/prepare.py   # 字符级编码 + meta.pkl(stoi/itos)
```

---

## 3. 顶层架构图

```mermaid
flowchart LR
    subgraph DATA["数据层 data/*/prepare.py"]
        RAW[("原始语料<br/>OpenWebText / Shakespeare")]
        TOK["tiktoken BPE<br/>或字符级 stoi/itos"]
        BIN[("train.bin / val.bin<br/>uint16 扁平数组")]
        META[("meta.pkl<br/>vocab_size / 编解码表")]
        RAW --> TOK --> BIN
        TOK --> META
    end

    subgraph CFG["配置层"]
        CLI["命令行 --key=value"]
        PRESET["config/*.py 预设"]
        CONFR["configurator.py<br/>exec 覆盖 globals()"]
        CLI --> CONFR
        PRESET --> CONFR
    end

    subgraph TRAIN["训练层 train.py"]
        LOADER["get_batch()<br/>np.memmap 随机切片"]
        INIT{"init_from"}
        SCRATCH["从零初始化"]
        RESUME["resume<br/>加载 ckpt.pt"]
        HF["gpt2*<br/>HF 权重导入"]
        OPT["AdamW + GradScaler"]
        LOOP["训练循环<br/>autocast + 梯度累积 + DDP"]
        CKPT[("out/ckpt.pt")]
        WANDB[("W&B 可选")]
        LOADER --> LOOP
        INIT --> SCRATCH & RESUME & HF
        SCRATCH & RESUME & HF --> LOOP
        OPT --> LOOP
        LOOP --> CKPT
        LOOP --> WANDB
    end

    subgraph MODEL["模型层 model.py"]
        GPTCONF["GPTConfig (dataclass)"]
        GPT["GPT 主模型"]
        BLOCK["Block × n_layer"]
        ATTN["CausalSelfAttention<br/>FlashAttention"]
        MLP["MLP<br/>4× GELU FFN"]
        GPTCONF --> GPT
        GPT --> BLOCK
        BLOCK --> ATTN & MLP
    end

    subgraph INFER["推理层 sample.py / bench.py"]
        SAMP["sample.py<br/>generate() 采样"]
        BENCH["bench.py<br/>性能基准"]
    end

    BIN --> LOADER
    META -.vocab_size.-> INIT
    CONFR -.覆盖参数.-> TRAIN
    CONFR -.覆盖参数.-> INFER
    INIT --> GPT
    LOOP -.forward/backward.-> GPT
    CKPT --> SAMP
    GPT --> BENCH
```

---

## 4. 模型结构（`model.py`）

### 4.1 类层级与组合关系

```mermaid
classDiagram
    class GPTConfig {
        +int block_size = 1024
        +int vocab_size = 50304
        +int n_layer = 12
        +int n_head = 12
        +int n_embd = 768
        +float dropout = 0.0
        +bool bias = True
    }

    class GPT {
        +GPTConfig config
        +ModuleDict transformer
        +Linear lm_head
        +forward(idx, targets) (logits, loss)
        +generate(idx, max_new_tokens, temperature, top_k)
        +configure_optimizers(wd, lr, betas, device_type)
        +crop_block_size(block_size)
        +from_pretrained(model_type) classmethod
        +estimate_mfu(fwdbwd_per_iter, dt)
        +get_num_params()
    }

    class Block {
        +LayerNorm ln_1
        +CausalSelfAttention attn
        +LayerNorm ln_2
        +MLP mlp
        +forward(x)
    }

    class CausalSelfAttention {
        +Linear c_attn (3*n_embd)
        +Linear c_proj
        +Dropout attn_dropout
        +Dropout resid_dropout
        +bool flash
        +forward(x)
    }

    class MLP {
        +Linear c_fc (4*n_embd)
        +GELU gelu
        +Linear c_proj
        +Dropout dropout
        +forward(x)
    }

    class LayerNorm {
        +Parameter weight
        +Parameter bias?
        +forward(input)
    }

    GPT *-- GPTConfig
    GPT "1" *-- "n_layer" Block : transformer.h
    GPT *-- LayerNorm : transformer.ln_f
    Block *-- LayerNorm : ln_1, ln_2
    Block *-- CausalSelfAttention
    Block *-- MLP

    note for GPT "transformer = ModuleDict\n  wte: Embedding(vocab, n_embd)\n  wpe: Embedding(block_size, n_embd)\n  drop: Dropout\n  h: ModuleList[Block × n_layer]\n  ln_f: LayerNorm\nlm_head.weight 与 wte.weight 共享"
```

关键设计点：

- **权重绑定**：`self.transformer.wte.weight = self.lm_head.weight`（`model.py:138`）。
- **残差缩放初始化**：所有以 `c_proj.weight` 结尾的层用 `std = 0.02/sqrt(2*n_layer)` 初始化（GPT-2 论文 Trick，`model.py:143-145`）。
- **优化器参数分组**：维度 ≥ 2 的张量做 weight decay，其余不做（`configure_optimizers`，`model.py:263-287`）。
- **`from_pretrained`**：从 HuggingFace `GPT2LMHeadModel` 拷贝权重，对 `Conv1D` 风格的四个权重做转置（`model.py:245`）。

### 4.2 单步前向数据流

```mermaid
flowchart TB
    IDX["idx: LongTensor (B, T)"] --> WTE["wte: 词嵌入<br/>(B, T, n_embd)"]
    POS["pos = arange(T)"] --> WPE["wpe: 位置嵌入<br/>(T, n_embd)"]
    WTE --> ADD((+))
    WPE --> ADD
    ADD --> DROP["dropout"]
    DROP --> BLK["transformer.h: Block × n_layer"]

    subgraph SingleBlock["单个 Block（Pre-LN 残差）"]
        X1["x"] --> LN1["LayerNorm ln_1"] --> ATTN["CausalSelfAttention"] --> R1((+))
        X1 --> R1
        R1 --> LN2["LayerNorm ln_2"] --> MLP["MLP (4x GELU FFN)"] --> R2((+))
        R1 --> R2
        R2 --> XOUT["x'"]
    end

    BLK --> LNF["LayerNorm ln_f"]
    LNF --> COND{"训练? targets 非空?"}
    COND -- "训练" --> LMHEAD1["lm_head(全部时间步)<br/>logits (B, T, V)"] --> LOSS["F.cross_entropy<br/>ignore_index=-1"]
    COND -- "推理" --> LMHEAD2["lm_head(仅末位)<br/>logits (B, 1, V)"] --> NONE["loss = None"]
```

### 4.3 `CausalSelfAttention` 内部

```mermaid
flowchart LR
    X["x: (B, T, C)"] --> CATTN["c_attn: Linear → 3C"]
    CATTN --> SPLIT["split → q, k, v"]
    SPLIT --> RESH["view + transpose<br/>(B, nh, T, hs)"]
    RESH --> BR{"hasattr SDPA?"}
    BR -- "是 (PyTorch≥2.0)" --> FLASH["scaled_dot_product_attention<br/>is_causal=True"]
    BR -- "否" --> MANUAL["q·kᵀ/√d<br/>+ tril mask<br/>+ softmax + dropout<br/>· v"]
    FLASH --> CONCAT["transpose + view<br/>(B, T, C)"]
    MANUAL --> CONCAT
    CONCAT --> CPROJ["c_proj: Linear"] --> RDROP["resid_dropout"] --> Y["y"]
```

---

## 5. 训练循环（`train.py`）

### 5.1 启动 / 初始化序列

```mermaid
sequenceDiagram
    autonumber
    participant CLI as 命令行/torchrun
    participant TR as train.py
    participant CFGR as configurator.py
    participant ENV as DDP env
    participant DAT as data/*.bin
    participant MOD as model.py
    participant HF as HuggingFace
    participant CKPT as out/ckpt.pt

    CLI->>TR: python train.py config/xx.py --key=val
    TR->>CFGR: exec(open('configurator.py').read())
    CFGR-->>TR: 覆盖 globals (n_layer, batch_size, ...)
    TR->>ENV: 读 RANK/LOCAL_RANK/WORLD_SIZE
    alt ddp 模式
        TR->>ENV: init_process_group(nccl)
        ENV-->>TR: device = cuda:LOCAL_RANK
    end
    TR->>DAT: np.memmap(train.bin, val.bin)
    alt init_from = scratch
        TR->>MOD: GPTConfig + GPT(...)
    else init_from = resume
        TR->>CKPT: torch.load(ckpt.pt)
        TR->>MOD: GPT(...) + load_state_dict
    else init_from = gpt2*
        TR->>MOD: GPT.from_pretrained(...)
        MOD->>HF: GPT2LMHeadModel.from_pretrained
        HF-->>MOD: 权重 → 拷贝 (转置 Conv1D)
    end
    TR->>MOD: configure_optimizers → AdamW
    opt block_size 缩小
        TR->>MOD: crop_block_size(...)
    end
    opt compile=True
        TR->>TR: model = torch.compile(model)
    end
    opt ddp
        TR->>TR: model = DDP(model, device_ids=[...])
    end
```

### 5.2 单步训练循环

```mermaid
flowchart TB
    START(["每个 iter_num"]) --> LR["按 cosine 调度计算 lr<br/>get_lr(iter_num) → param_groups"]
    LR --> EVAL{"iter % eval_interval == 0<br/>且 master?"}
    EVAL -- "是" --> ESTLOSS["estimate_loss()<br/>train/val 各 eval_iters 个 batch"]
    ESTLOSS --> SAVE{"val 改善 或<br/>always_save_checkpoint?"}
    SAVE -- "是" --> WRITE[("torch.save → ckpt.pt")]
    SAVE -- "否" --> SKIPSAVE[" "]
    EVAL -- "否" --> ACCUM
    WRITE --> ACCUM
    SKIPSAVE --> ACCUM

    subgraph ACCUM["梯度累积 micro_step in range(grad_accum)"]
        DDP_SYNC["DDP: require_backward_grad_sync<br/>仅最后一次 micro_step 同步"]
        DDP_SYNC --> AUTOCAST["with ctx (autocast bf16/fp16):<br/>logits, loss = model(X, Y)<br/>loss /= grad_accum"]
        AUTOCAST --> PREFETCH["X, Y = get_batch('train')<br/>(异步预取下一个 batch)"]
        PREFETCH --> BWD["scaler.scale(loss).backward()"]
    end

    ACCUM --> CLIP{"grad_clip != 0?"}
    CLIP -- "是" --> UNSCALE["scaler.unscale_(optimizer)<br/>clip_grad_norm_"]
    CLIP -- "否" --> STEPOPT
    UNSCALE --> STEPOPT["scaler.step(optimizer)<br/>scaler.update()"]
    STEPOPT --> ZERO["optimizer.zero_grad(set_to_none=True)"]
    ZERO --> LOG["log_interval: estimate_mfu + 打印 / W&B"]
    LOG --> COND{"iter_num > max_iters?"}
    COND -- "否" --> START
    COND -- "是" --> END(["destroy_process_group"])
```

要点：

- **梯度累积 + DDP**：通过手动开关 `model.require_backward_grad_sync` 跳过中间 micro-step 的 all-reduce，只在最后一步同步（`train.py:292-305`），避免 `no_sync()` 上下文。
- **异步取批**：当前 step 在 GPU 上做前向时，CPU 端立刻预取下一个 batch（pin_memory + `non_blocking=True`，`train.py:127-131,303`）。
- **GradScaler**：仅当 `dtype == 'float16'` 时真正生效，`bfloat16` 下是 no-op。
- **MFU 估计**：依据 PaLM 论文 Appendix B 的公式 `flops_per_token = 6N + 12LHQT`，归一化到 A100 bf16 峰值 312 TFLOPS（`model.py:289-303`）。

---

## 6. 数据准备流（`data/*/prepare.py`）

```mermaid
flowchart LR
    subgraph OWT["openwebtext (生产数据集)"]
        A1["load_dataset('openwebtext')<br/>HuggingFace datasets"]
        A2["train_test_split(0.0005)<br/>→ train / val"]
        A3["tiktoken gpt2 BPE<br/>+ 追加 eot_token"]
        A4["分 1024 shard 写入<br/>np.memmap uint16"]
        A1 --> A2 --> A3 --> A4
        A4 --> A5[("train.bin ~17GB<br/>val.bin ~8.5MB")]
    end

    subgraph SHK["shakespeare (BPE)"]
        B1["下载 tinyshakespeare"] --> B2["90/10 划分"] --> B3["tiktoken BPE"]
        B3 --> B4[("train.bin / val.bin")]
    end

    subgraph SHKC["shakespeare_char (字符级)"]
        C1["下载 tinyshakespeare"] --> C2["构造 stoi/itos"] --> C3["字符 → int"]
        C3 --> C4[("train.bin / val.bin")]
        C2 --> C5[("meta.pkl<br/>vocab_size=65, stoi, itos")]
    end

    A5 & B4 & C4 --> TRAINING["train.py get_batch()"]
    C5 -.train 读取 vocab_size.-> TRAINING
```

`get_batch(split)` 的核心是：

```
data = np.memmap(...)
ix   = randint(len(data) - block_size, (batch_size,))
x    = stack(data[i : i+block_size])      # 输入
y    = stack(data[i+1 : i+1+block_size])  # 右移一位的标签
```

即每个 batch 都从全量 token 流中随机抽 `batch_size` 个 `block_size` 长的窗口，标签是输入右移一位，天然实现 teacher forcing。

---

## 7. 推理 / 采样（`sample.py`）

```mermaid
sequenceDiagram
    autonumber
    participant U as 用户
    participant S as sample.py
    participant CKPT as out/ckpt.pt
    participant META as data/.../meta.pkl
    participant TIK as tiktoken
    participant M as GPT

    U->>S: python sample.py --start="..." --num_samples=N
    alt init_from = resume
        S->>CKPT: torch.load(...)
        S->>M: GPT(GPTConfig(**ckpt['model_args']))
        S->>M: load_state_dict (去 _orig_mod 前缀)
    else init_from = gpt2*
        S->>M: GPT.from_pretrained(...)
    end
    S->>M: model.eval().to(device)

    alt 训练时保留 meta.pkl (字符级)
        S->>META: pickle.load → stoi/itos
    else GPT-2 BPE 默认路径
        S->>TIK: get_encoding('gpt2')
    end

    S->>S: 编码 prompt → idx (1, t)
    loop num_samples 次
        S->>M: model.generate(idx, max_new_tokens, temperature, top_k)
        loop max_new_tokens 步
            M->>M: 裁剪 idx 到 block_size
            M->>M: forward → 取最后一步 logits
            M->>M: /temperature + top_k 截断
            M->>M: softmax + multinomial 采样
            M->>M: 拼接新 token
        end
        M-->>S: 完整 token 序列
        S->>U: 解码并打印
    end
```

`bench.py` 与 `train.py` 几乎同构，但 (1) 不写检查点 / 不做 eval；(2) 可选 `torch.profiler` 导出 TensorBoard trace；(3) 用前 10 步预热、后 20 步计时，输出 `time/iter` 与 MFU。

---

## 8. 配置覆盖机制（`configurator.py`）

```mermaid
flowchart LR
    A["python train.py<br/>config/x.py --batch_size=32"] --> B["train.py 顶部声明默认变量<br/>(模块级 globals)"]
    B --> C["exec(open('configurator.py').read())"]
    C --> D{"sys.argv[i]"}
    D -- "无 '='" --> E["视为文件路径<br/>exec(open(arg).read())"]
    D -- "--key=val" --> F["literal_eval(val)<br/>类型必须匹配 globals[key]"]
    E --> G["覆盖 globals()"]
    F --> G
    G --> H["后续训练代码读取最终的<br/>n_layer, batch_size, dataset, ..."]
```

这是 nanoGPT 最有争议但也最简洁的设计：所有配置项就是脚本顶部的普通 Python 变量，`configurator.py` 通过 `exec` 直接改写宿主脚本的 `globals()`，因此无需任何 `config.xxx` 前缀。

---

## 9. 三类典型运行路径

| 场景 | 启动 | 关键路径 |
|------|------|----------|
| 从零训练 GPT-2 124M | `torchrun --standalone --nproc_per_node=8 train.py config/train_gpt2.py` | `init_from='scratch'` → 新建 GPT → AdamW → 600k iter / 300B tokens |
| 字符级 Shakespeare 玩具 | `python train.py config/train_shakespeare_char.py` | `meta.pkl` 提供 `vocab_size=65` → 6 层小模型 → 5000 iter |
| 微调 GPT-2-XL | `python train.py config/finetune_shakespeare.py` | `init_from='gpt2-xl'` → HF 权重导入 → 常数 LR 微调 20 iter |
| 采样 | `python sample.py --out_dir=out-shakespeare-char --start="ROMEO:"` | 加载 ckpt → `generate()` 自回归 |
| 评估预训练 GPT-2 | `python train.py config/eval_gpt2.py` | `eval_only=True` → 仅跑一次 `estimate_loss` |
| 性能基准 | `python bench.py` | 不存盘 / 不做 DDP / 可选 profiler |

---

## 10. 关键交互一句话总结

> **`configurator.py`** 把命令行和 `config/*.py` 注入 **`train.py`** 的 `globals`；**`train.py`** 用 **`data/*/prepare.py`** 写出的 **`train.bin`** 通过 **`np.memmap`** 喂数据，把 **`model.py`** 的 **`GPT`**（由 **`Block`**、**`CausalSelfAttention`**、**`MLP`** 组成）放进 **AMP + DDP + torch.compile** 的训练环里，定期写出 **`out/ckpt.pt`**；**`sample.py`** 读回该 checkpoint 调 **`GPT.generate`** 完成自回归采样；**`bench.py`** 则跑同样的 forward/backward 用于性能基准。
