# LangChain 项目架构与核心模块交互

## 一、分层架构总览

```mermaid
graph TD
    APP["Application Layer<br/>用户代码 / LangGraph Agent / RAG Pipeline"]
    LC["langchain (v1) — 便捷层<br/>init_chat_model() / init_embeddings() / agents"]
    OAI["langchain-openai<br/>ChatOpenAI / OpenAIEmbeddings"]
    ANT["langchain-anthropic<br/>ChatAnthropic / convert_to_anthropic_tool()"]
    TS["langchain-text-splitters<br/>RecursiveCharacterTextSplitter / MarkdownHeaderTextSplitter"]
    CORE["langchain-core — 基础层"]

    APP --> LC
    LC --> OAI
    LC --> ANT
    LC --> TS
    OAI --> CORE
    ANT --> CORE
    TS --> CORE

    subgraph CORE_MODULES [langchain-core 内部模块]
        direction LR
        Runnables
        Messages
        Tools
        Prompts
        OutputParsers["Output Parsers"]
        Documents
        Retrievers
        VectorStore
        Embeddings
        Callbacks
        Utils["Utils<br/>function_calling"]
    end

    CORE --- CORE_MODULES
```

---

## 二、Runnable — 一切的基石

`Runnable` 是整个 LangChain 的核心抽象。几乎所有组件都继承自它，使得任何组件都能通过 `|` 管道符组合。

```mermaid
classDiagram
    class Runnable {
        <<abstract>>
        +invoke(input) Output
        +ainvoke(input) Output
        +batch(inputs) list~Output~
        +stream(input) Iterator~Output~
        +astream(input) AsyncIterator~Output~
    }

    class RunnableSerializable {
        +to_json() dict
    }

    class RunnableSequence {
        A | B | C 管道
    }

    class RunnableParallel {
        key1: A, key2: B 并行
    }

    class RunnableLambda {
        包装普通函数
    }

    class RunnableGenerator {
        包装生成器 - 流式
    }

    class RunnablePassthrough {
        数据透传
    }

    class BasePromptTemplate {
        dict → PromptValue
    }

    class BaseChatModel {
        messages → AIMessage
    }

    class BaseOutputParser {
        AIMessage → 结构化数据
    }

    class BaseTool {
        str|dict|ToolCall → Any
    }

    class BaseRetriever {
        str → list~Document~
    }

    Runnable <|-- RunnableSerializable
    Runnable <|-- RunnableLambda
    Runnable <|-- RunnableGenerator

    RunnableSerializable <|-- RunnableSequence
    RunnableSerializable <|-- RunnableParallel
    RunnableSerializable <|-- RunnablePassthrough
    RunnableSerializable <|-- BasePromptTemplate
    RunnableSerializable <|-- BaseLanguageModel
    RunnableSerializable <|-- BaseOutputParser
    RunnableSerializable <|-- BaseTool
    RunnableSerializable <|-- BaseRetriever

    BaseLanguageModel <|-- BaseChatModel
    BaseLanguageModel <|-- BaseLLM

    BaseChatModel <|-- ChatOpenAI
    BaseChatModel <|-- ChatAnthropic

    BasePromptTemplate <|-- PromptTemplate
    BasePromptTemplate <|-- ChatPromptTemplate

    BaseTool <|-- StructuredTool
    BaseTool <|-- Tool

    BaseRetriever <|-- VectorStoreRetriever
```

**统一接口**：所有 Runnable 都实现这些方法：

| 方法 | 说明 |
|------|------|
| `invoke(input)` | 单次同步调用 |
| `ainvoke(input)` | 单次异步调用 |
| `batch(inputs)` | 批量调用 |
| `stream(input)` | 流式输出 |
| `astream(input)` | 异步流式 |

---

## 三、核心数据流

### 1. 基本对话链

```mermaid
flowchart LR
    A["dict"] -->|输入变量| B["ChatPromptTemplate"]
    B -->|PromptValue| C["BaseChatModel<br/>(ChatOpenAI / ChatAnthropic)"]
    C -->|AIMessage| D["BaseOutputParser"]
    D -->|T| E["结构化结果"]

    style A fill:#f9f,stroke:#333
    style E fill:#9f9,stroke:#333
```

> **LCEL 写法**: `chain = prompt | model | parser`

### 2. Tool Calling 循环

```mermaid
flowchart TD
    A["Messages"] --> B["BaseChatModel<br/>.bind_tools()"]
    B --> C["AIMessage<br/>.tool_calls"]
    C --> D{有 tool calls?}
    D -->|No| E["返回最终结果"]
    D -->|Yes| F["BaseTool.invoke()"]
    F --> G["ToolMessage"]
    G -->|结果回传给模型| A
```

### 3. Tool Schema 转换流程

```mermaid
flowchart TD
    I1["@tool 装饰器"] --> CONV
    I2["Pydantic BaseModel"] --> CONV
    I3["Python Callable"] --> CONV
    I4["BaseTool 实例"] --> CONV
    I5["dict"] --> CONV

    CONV["convert_to_openai_tool()<br/>统一中间表示 IR"]

    CONV --> P1["OpenAI: 直接使用"]
    CONV --> P2["Anthropic: convert_to_anthropic_tool()<br/>字段映射"]
    CONV --> P3["其他 Provider: 各自映射"]
```

### 4. RAG 检索链

```mermaid
flowchart TD
    subgraph 索引阶段
        RAW["原始文本"] --> SPLIT["TextSplitter"]
        SPLIT --> DOCS["list&lt;Document&gt;"]
        DOCS --> EMB1["Embeddings.embed_documents()"]
        EMB1 --> VS_ADD["VectorStore.add_texts()<br/>存储向量"]
    end

    subgraph 查询阶段
        QUERY["用户 query"] --> EMB2["Embeddings.embed_query()"]
        EMB2 --> VS_SEARCH["VectorStore<br/>similarity_search()"]
        VS_SEARCH --> RESULTS["list&lt;Document&gt;"]
        RESULTS --> PROMPT["注入 Prompt 上下文"]
        PROMPT --> MODEL["ChatModel 生成回答"]
    end

    VS_ADD -.->|同一个 VectorStore| VS_SEARCH
```

---

## 四、消息体系

```mermaid
classDiagram
    class BaseMessage {
        +content: str | list
        +additional_kwargs: dict
        +response_metadata: dict
        +type: str
        +name: str | None
        +id: str | None
    }

    class HumanMessage {
        type = "human"
        用户输入
    }

    class SystemMessage {
        type = "system"
        系统指令
    }

    class AIMessage {
        type = "ai"
        +tool_calls: list~ToolCall~
        +invalid_tool_calls: list
        +usage_metadata: UsageMetadata
    }

    class ToolMessage {
        type = "tool"
        +tool_call_id: str
        +status: success | error
    }

    class FunctionMessage {
        type = "function"
        遗留 - OpenAI function calling
    }

    class RemoveMessage {
        消息历史管理标记
    }

    class ToolCall {
        +name: str
        +args: dict
        +id: str | None
    }

    BaseMessage <|-- HumanMessage
    BaseMessage <|-- SystemMessage
    BaseMessage <|-- AIMessage
    BaseMessage <|-- ToolMessage
    BaseMessage <|-- FunctionMessage
    BaseMessage <|-- RemoveMessage

    AIMessage *-- ToolCall : tool_calls
    ToolMessage --> ToolCall : tool_call_id 关联
```

---

## 五、回调系统 — 贯穿全局的观测层

```mermaid
flowchart TD
    RC["RunnableConfig<br/>{ callbacks: [...] }"]
    RC --> CM["CallbackManager"]

    CM --> LLM_E["LLM 事件<br/>on_llm_start / end / error<br/>on_llm_new_token"]
    CM --> CHAIN_E["Chain 事件<br/>on_chain_start / end / error"]
    CM --> TOOL_E["Tool 事件<br/>on_tool_start / end / error"]
    CM --> RET_E["Retriever 事件<br/>on_retriever_start / end / error"]
    CM --> CUSTOM["on_custom_event"]

    subgraph 内置 Handler
        H1["StdOutCallbackHandler<br/>控制台输出"]
        H2["StreamingStdOutCallbackHandler<br/>流式输出"]
        H3["FileCallbackHandler<br/>写文件"]
        H4["UsageMetadataCallbackHandler<br/>token 统计"]
    end

    CM --> H1
    CM --> H2
    CM --> H3
    CM --> H4
```

每个 Runnable 的 `invoke/stream/batch` 调用时，都会将 `callbacks` 通过 `RunnableConfig` 传播给下游，形成完整的调用链追踪。

---

## 六、Partner 集成模式

```mermaid
flowchart LR
    subgraph "langchain-core（接口定义）"
        BCM["BaseChatModel<br/>_generate() / _stream()<br/>bind_tools()"]
        EMB["Embeddings<br/>embed_query() / embed_documents()"]
    end

    subgraph "langchain-openai"
        CO["ChatOpenAI<br/>调用 OpenAI API"]
        OE["OpenAIEmbeddings"]
    end

    subgraph "langchain-anthropic"
        CA["ChatAnthropic<br/>调用 Anthropic API"]
    end

    subgraph "langchain-tests"
        ST["ChatModelTests<br/>EmbeddingsTests<br/>标准测试基类"]
    end

    BCM -->|extends| CO
    BCM -->|extends| CA
    EMB -->|extends| OE

    ST -.->|验证接口兼容| CO
    ST -.->|验证接口兼容| CA
    ST -.->|验证接口兼容| OE
```

---

## 七、模块依赖关系总图

```mermaid
flowchart TD
    subgraph CORE["langchain-core"]
        CB["Callbacks"] -.->|贯穿所有组件| RUN["Runnable<br/>统一执行协议"]

        RUN --- PROMPT["Prompts"]
        RUN --- LM["Language Models"]
        RUN --- OP["Output Parsers"]
        RUN --- TOOL["Tools"]
        RUN --- RET["Retrievers"]

        PROMPT -->|PromptValue| LM
        LM -->|AIMessage| OP
        OP -->|结构化数据| OUT["输出"]

        LM -->|AIMessage.tool_calls| TOOL
        TOOL -->|ToolMessage| LM

        DOC["Documents"] --> EMB["Embeddings"]
        EMB --> VS["VectorStore"]
        VS --> VSR["VectorStoreRetriever"]
        VSR --> RET

        UTIL["Utils<br/>convert_to_openai_tool()<br/>tool schema 中间表示"]
        UTIL -.-> TOOL
    end

    OAI["langchain-openai<br/>具体实现"] --> CORE
    ANT["langchain-anthropic<br/>具体实现"] --> CORE
    LCV1["langchain (v1)<br/>工厂 + 便捷 API"] --> CORE
    TXT["langchain-text-splitters"] --> DOC
```

---

## 核心设计思想

1. **Runnable 统一协议** — 一切皆 Runnable，通过 `|` 组合，自动获得 sync/async/stream/batch 能力
2. **分层解耦** — core 定义接口，partner 实现细节，langchain 提供便捷入口
3. **消息驱动** — 组件间通过类型化的 Message 对象通信，tool call 也内嵌在 AIMessage 中
4. **回调观测** — Callbacks 通过 RunnableConfig 自动传播，实现全链路可观测