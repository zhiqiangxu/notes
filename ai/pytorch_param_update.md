# 参数更新过程：演绎式推导

> 目标：从"我们要训练模型"这一最初动机出发，每一步严格从上一步推出，
> 直到搞清楚 PyTorch 里每一次 `optimizer.step()` 到底改了什么、为什么这么改。

---

## 0. 出发点：训练到底要做什么

**前提：** 模型有一堆参数 `θ = (W₁, b₁, W₂, b₂, …)`。给定一批数据，模型按
当前 `θ` 算出 `loss(θ; data)`——一个**标量**。

**训练目标：** 找到一个 `θ` 让 `loss` 尽量小。

**问题转化：** "如何调整 `θ` 让 `loss` 变小" 是一个**优化问题**。

**结论 0：** 训练就是优化。每一次"更新参数"就是优化过程的一步。

---

## 1. 优化的基本工具：梯度

**前提：** `loss` 关于每个参数 `p` 的偏导 `∂loss/∂p` 衡量"`p` 改变一点点时
`loss` 的瞬时变化率"。

**推论 1.1：** 如果把所有 `∂loss/∂p` 收集起来，就得到一个和 `p` 同形状的张量
`g = ∇loss`，它指向**`p` 空间里 loss 上升最快的方向**。

**推论 1.2：** **`-g` 就是 loss 下降最快的方向**。

**梯度下降的基本更新规则：**

```
p_new  =  p  -  lr · g
```

- `lr`（学习率）：步长，控制每次走多远。
- 减号：把"上升方向"翻成"下降方向"。

**结论 1：** 只要能算出每个参数的 `g`，我们就有了一个原则上有效的更新规则。

> 这也回答了"为什么是 `∂loss/∂p` 而不是 `∂p/∂loss`"——只有前者有定义，
> 而且只有前者能告诉我们该往哪边推参数。

---

## 2. 怎么把 `g` 算出来：反向传播

**前提：** 现代模型有几亿到几千亿个参数。每个参数的 `∂loss/∂p` 都得算出来。
直接对每个 `p` 单独求导是不可承受的。

**关键算法：** **反向传播 (backpropagation)** = 链式法则在计算图上的高效实现。

### 2.1 前向时构建计算图

每当一个 op 被执行（比如 `out = A @ B`），PyTorch 在 `out` 上挂一个
`grad_fn` 节点（这里是 `MmBackward0`），记录：

- **这个 op 是哪个 op**（用于稍后调用它的反向公式）。
- **它的输入是谁**（即上游连接的节点）。
- **反向时需要的中间张量**（这里是 `A` 和 `B` 本身，用于算 VJP）。

整个网络的所有 op 这样连起来构成一个 DAG，叶子就是参数（`requires_grad=True`
的 `Parameter` 张量）。

### 2.2 反向时沿图反向走

`loss.backward()` 的算法：

1. **种子梯度：** `g_loss = ∂loss/∂loss = 1`。
2. **拓扑反向遍历：** 按反向拓扑序走计算图。
3. **每个节点调用自己的反向公式 (VJP)：** 给定从下游来的 `g_out`，输出对每个输入的 `g_input`。
   - 例：`mm` 节点收到 `g_out`，输出 `g_A = g_out @ Bᵀ` 和 `g_B = Aᵀ @ g_out`。
4. **到达叶子参数时累加到 `.grad`：** `p.grad += g_p`。
   - 这是由叶子节点的特殊 op `AccumulateGrad` 完成的。

**结论 2：** `.backward()` 结束后，每个参与 loss 计算的叶子参数 `p` 的 `.grad`
字段里都已经存好了 `∂loss/∂p`。

---

## 3. `.grad` 是累加的，不是覆盖的

**前提：** 第 2 步说每次 `.backward()` 会做 `p.grad += g_p`，**不是** `p.grad = g_p`。

**为什么这么设计？** 因为这让以下两个常见场景**直接可用**：

### 3.1 梯度累积 (gradient accumulation)

显卡装不下大 batch 时，把大 batch 拆成 K 个微批：

```python
optimizer.zero_grad()
for k in range(K):
    loss_k = compute(microbatch_k) / K
    loss_k.backward()          # K 次累加进同一个 .grad
optimizer.step()               # 用累加好的总梯度更新一次
```

**数学上恰好等价于一次大 batch 的梯度**，因为：

```
g_total = Σ_k g_k  =  Σ_k (1/K) · ∂loss_k/∂p  =  ∂(mean loss)/∂p
```

### 3.2 多目标 loss

```python
loss_a.backward(retain_graph=True)   # p.grad = ∇loss_a
loss_b.backward()                    # p.grad = ∇loss_a + ∇loss_b
optimizer.step()                     # 一起更新
```

**结论 3：** "累加"是基础原语；如果想从零开始一次新的梯度，必须显式调
`optimizer.zero_grad()`。

---

## 4. 拿到 `.grad` 后，optimizer 如何更新 `.data`

**前提：** `.backward()` 完成后，每个叶子参数 `p` 有两个张量字段：

- `p.data`：当前权重值。
- `p.grad`：刚算好的梯度。

**`optimizer.step()` 的工作：** 遍历它管理的每个参数 `p`，按预设的"更新规则"
读 `p.grad`、读自己维护的"优化器状态"（如果有）、然后**原地修改 `p.data`**。

不同的 optimizer 只是"更新规则"不同。下面把三个常见的展开。

### 4.1 SGD（最简单）

```
p.data  ←  p.data  -  lr · p.grad
```

- 无状态。
- 每个参数被一视同仁地按当前梯度方向走 `lr` 步。

### 4.2 SGD with momentum

```
v       =  β · v  +  p.grad           ← 状态：速度
p.data  ←  p.data  -  lr · v
```

- 每个参数一份 `v`，记录梯度的指数滑动平均。
- 在持续同向的梯度上"加速"，在震荡的梯度上"抵消"。
- 典型 `β = 0.9`。

### 4.3 AdamW（LLM 训练的事实标准）

```
m       =  β₁ · m  +  (1 - β₁) · p.grad           ← 一阶矩（梯度均值）
v       =  β₂ · v  +  (1 - β₂) · p.grad²          ← 二阶矩（梯度方差代理）

m_hat   =  m / (1 - β₁ ** t)                       ← 偏差修正
v_hat   =  v / (1 - β₂ ** t)

p.data  ←  p.data  -  lr · weight_decay · p.data   ← decoupled weight decay
p.data  ←  p.data  -  lr · m_hat / (sqrt(v_hat) + eps)
```

每个参数维护 `m, v` 两个状态张量。直觉：

- `m_hat` 提供平滑后的**更新方向**。
- `sqrt(v_hat)` 提供**逐参数自适应的步长缩放**——梯度大幅波动的参数自动用小步，稳定的参数用大步。
- `weight_decay` 项独立于梯度，把权重朝 0 拉一点（正则化）。

典型超参：`lr=1e-4 ~ 1e-3`，`β₁=0.9`，`β₂=0.95 ~ 0.999`，`eps=1e-8`，`weight_decay=0.01 ~ 0.1`。

**结论 4：** optimizer **只读 `p.grad`、只写 `p.data` 和自己的状态**。它对
模型结构、loss 形式、`p` 是怎么来的一无所知——只看 `(p.data, p.grad, state[p])`
这个三元组。

---

## 5. 清梯度，准备下一轮

**前提：** 第 3 步说 `.grad` 是累加的。这一步的 `.grad` 用过之后必须清零，
否则下一轮 `.backward()` 会叠加到旧值上，等于在错误的方向走。

```python
optimizer.zero_grad()    # 或 optimizer.zero_grad(set_to_none=True)
```

它对每个被管理的参数做：

- `set_to_none=True`（现代默认）：`p.grad = None`，下一次 backward 重新分配。
- `set_to_none=False`：`p.grad.zero_()`，原地填零。

**注意：** 这只清 `.grad`，**不**动 `.data`、**不**动 optimizer 状态 `m, v`、
**不**重置学习率。

**结论 5：** 一次完整的参数更新 = backward 算梯度 → step 用梯度改权重 →
zero_grad 清梯度。三步缺一不可。

---

## 6. 完整时间线（一次训练迭代）

```
forward:        h = model(x)
                loss = criterion(h, y)
                            ↓
                            构建计算图（每个张量挂 grad_fn）

backward:       loss.backward()
                            ↓
                            从 loss 反向遍历图
                            每个 op 调用反向公式 (VJP)
                            到达叶子 p 时：p.grad += ∂loss/∂p

(可选) 梯度裁剪: torch.nn.utils.clip_grad_norm_(params, max_norm)
                            ↓
                            原地缩放每个 p.grad，使全局 L2 范数 ≤ max_norm

step:           optimizer.step()
                            ↓
                            对每个 p：
                              更新 state[p]（如 m, v）
                              p.data ← update_rule(p.data, p.grad, state[p])

清梯度:         optimizer.zero_grad()
                            ↓
                            p.grad = None  （或填零）

(进入下一个 iteration)
```

`p.data` **在 step 时被原地修改**。这是它这一步**唯一**发生变化的地方。
下一次 forward 用的就是新的 `p.data`，所以下一次的 loss、梯度、更新都跟着不同。

---

## 7. 梯度累积的完整版本（带累加步数）

```python
optimizer.zero_grad()

for k in range(gradient_accumulation_steps):
    x_k, y_k = next(dataloader)
    log_probs = get_response_log_probs(model, x_k, y_k, False)["log_probs"]
    loss, _ = sft_microbatch_train_step(
        log_probs,
        response_mask_k,
        gradient_accumulation_steps=K,   # 内部除以 K
        normalize_constant=C,
    )
    # loss.backward() 已经在函数内部执行

# K 次 backward 累加完毕，p.grad = 大 batch 的等价梯度

torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
optimizer.step()
optimizer.zero_grad()
```

注意 `zero_grad` 在**整个 K 步循环之前和之后**各一次，**绝不能**放进循环内——
那会把累积好的梯度清掉。

---

## 8. 各阶段所改变的状态汇总

| 阶段 | 改变 `p.data`？ | 改变 `p.grad`？ | 改变 optimizer 状态 `m,v`？ |
|---|---|---|---|
| forward | 否 | 否 | 否 |
| `loss.backward()` | 否 | **是**（`+=`） | 否 |
| `clip_grad_norm_` | 否 | **是**（原地缩放） | 否 |
| `optimizer.step()` | **是** | 否 | **是** |
| `optimizer.zero_grad()` | 否 | **是**（清零或置 None） | 否 |

> **核心分工：**
> - 计算梯度 → `.backward()` 写 `.grad`
> - 应用更新 → `optimizer.step()` 用 `.grad` 改 `.data`
> - 它们靠 `.grad` 这个共享字段通信，彼此互不依赖

---

## 9. 一个完整的数值小例子

设：
```
A.data = [[1.0, 2.0]]      形状 (1, 2)
B.data = [[3.0], [4.0]]    形状 (2, 1)
loss = A @ B               = 11
lr = 0.1                   用 SGD
```

**Forward：** `loss = 11`。

**Backward：** 上游 `g_loss = 1`。

```
g_A = g_loss @ Bᵀ  =  1 · [[3, 4]]      = [[3, 4]]
g_B = Aᵀ @ g_loss  =  [[1], [2]] · 1   = [[1], [2]]
```

`AccumulateGrad` 把它们写进 `.grad`：

```
A.grad = [[3, 4]]
B.grad = [[1], [2]]
```

**Step（SGD）：**

```
A.data ← [[1, 2]] - 0.1 · [[3, 4]]    = [[0.7, 1.6]]
B.data ← [[3], [4]] - 0.1 · [[1], [2]] = [[2.9], [3.8]]
```

**验证 loss 是否下降：** 用新 `A.data, B.data` 重算：

```
new loss  =  [[0.7, 1.6]] @ [[2.9], [3.8]]
          =  0.7 · 2.9 + 1.6 · 3.8
          =  2.03 + 6.08
          =  8.11
```

`11 → 8.11`。loss 真的减小了——更新方向正确。

**清梯度：**

```
A.grad = None
B.grad = None
```

下一轮 forward 用新的 `A.data, B.data`，故事重新开始。

---

## 10. 一句话总结

> 训练 = 重复执行 `forward → backward → step → zero_grad`。
>
> - `forward` 算 loss 并搭建计算图。
> - `backward` 沿图反向把 loss 分摊回每个参数，结果累加进 `.grad`。
> - `step` 读 `.grad`、套用 optimizer 自己的更新规则、原地改 `.data`。
> - `zero_grad` 清掉这一步的 `.grad`，准备下一次累加。
>
> 参数之所以会被更新成"减小 loss 的方向"，是因为梯度本身指向"loss 增大最快"
> 的方向，而 optimizer 把它**反着用**。所有复杂度（动量、自适应步长、权重衰减）
> 都是在这条"反着用梯度"的核心思想上加修饰。
