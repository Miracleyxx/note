## **BMF (Babit Multimedia Framework) 深度解析学习笔记 (图文详解版)**

### **1. 引言与概述**

BMF 是一个高性能、模块化的流媒体处理框架。本笔记结合源码分析和图示，旨在清晰地展示其核心架构和工作原理。

**核心目标:** 实现高效、灵活、可扩展的流式多媒体处理。

### **2. 核心概念 (图示)**

Code snippet

```mermaid
graph LR
    subgraph BMF Core Concepts
        G[Graph 图<br/>处理流程蓝图] --- N[Node 节点<br/>执行单元]
        N --- M[Module 模块<br/>具体处理逻辑]
        N --- ISM[InputStreamManager<br/>输入流管理/同步]
        N --- OSM[OutputStreamManager<br/>输出流管理/分发]
        ISM --- IS[InputStream<br/>输入流缓冲区]
        OSM --- OS[OutputStream<br/>输出流连接点]
        IS --- P[Packet 数据包<br/>VideoFrame/AudioFrame]
        OS --- P
        Scheduler --- T[Task 任务<br/>调度单元/数据载体]
        T --- N
        T --- P
    end
    style G fill:#f9f
    style N fill:#ccf
    style M fill:#ffc
    style ISM fill:#9cf
    style OSM fill:#9cf
    style IS fill:#c9f
    style OS fill:#c9f
    style P fill:#fcc
    style T fill:#9fc
    style Scheduler fill:#ff9
```

### **3. 整体架构层次 (图示)**

BMF 采用清晰的分层设计：

Code snippet

```mermaid
graph TD
    A["应用层 (Application Layer)<br/>业务逻辑, 使用SDK构建图"] --> B["API层<br/>Python/C++/Go SDK, Graph Builder"]
    B --> C["图引擎层 (Graph Engine)<br/>Graph, Node, Stream Management"]
    C --> D["调度器层 (Scheduler Layer)<br/>Scheduler, SchedulerQueue, Task"]
    D --> E["模块层 (Module Layer)<br/>Module Factory, Module Interface"]
    E --> F["数据处理层 (Data Processing)<br/>Packet, Memory Management, Data Conversion"]

    style A fill:#f9f,stroke:#333,stroke-width:2px
    style B fill:#ccf,stroke:#333,stroke-width:2px
    style C fill:#9cf,stroke:#333,stroke-width:2px
    style D fill:#9fc,stroke:#333,stroke-width:2px
    style E fill:#ffc,stroke:#333,stroke-width:2px
    style F fill:#fcc,stroke:#333,stroke-width:2px
```



### **4. 核心组件交互关系 (图示)**

这张图展示了主要组件如何协同工作：

Code snippet

```mermaid
graph TB
    subgraph "Graph (图引擎)"
        Graph[Graph 实例]
        Graph -- Manages --> NodesMap{{"Node Map<br/>(id -> Node*) "}}
        Graph -- Owns --> Scheduler[Scheduler]
        Graph -- Manages --> GI[Graph Input Streams]
        Graph -- Manages --> GO[Graph Output Streams]
    end

    subgraph "Node (节点, e.g., Node X)"
        NodeX[Node X 实例]
        NodeX -- Owns --> ModuleX[Module X 实例]
        NodeX -- Owns --> ISMX[InputStreamManager X]
        NodeX -- Owns --> OSMX[OutputStreamManager X]
        NodesMap -- Contains --> NodeX
    end

    subgraph "Stream Managers"
        ISMX -- Manages --> ISMapX{{"InputStream Map<br/>(id->IS*) "}}
        OSMX -- Manages --> OSMapX{{"OutputStream Map<br/>(id->OS*) "}}
    end

    subgraph "Streams"
        ISX0[InputStream X.0] --> QX0["SafeQueue&lt;Packet&gt;"]
        ISX1[InputStream X.1] --> QX1["SafeQueue&lt;Packet&gt;"]
        OSX0[OutputStream X.0] -- Has --> MirrorsX0[Mirror List]
        OSX1[OutputStream X.1] -- Has --> MirrorsX1[Mirror List]
        ISMapX -- Contains --> ISX0 & ISX1
        OSMapX -- Contains --> OSX0 & OSX1
    end

    subgraph "Scheduler (调度器)"
        Scheduler -- Manages --> SQMap{{"SchedulerQueue Map<br/>(id->SQ*) "}}
        Scheduler -- Manages --> NodesToSched{{"Nodes To Schedule<br/>Map (id->NodeItem)"}}
        SQ[SchedulerQueue Y] -- Owns --> PQ["SafePriorityQueue&lt;Item&gt;"]
        SQ -- Owns --> WorkerThread[Worker Thread]
        SQMap -- Contains --> SQ
    end

    subgraph "Task (任务)"
        TaskInst[Task 实例]
        TaskInst -- Contains --> InQ["Input Queues<br/>(id->queue*)"]
        TaskInst -- Contains --> OutQ["Output Queues<br/>(id->queue*)"]
        TaskInst -- Belongs to --> NodeIDInfo["Node ID"]
        TaskInst -- Carries --> TimestampInfo["Timestamp"]
    end
```

```mermaid
graph TD
    %% --- 1. 定义交互涉及的核心组件 ---
    GI[Graph Input Streams]
    GO[Graph Output Streams]
    ISMX[InputStreamManager X]
    TaskInst[Task 实例]
    Scheduler[Scheduler]
    PQ["SafePriorityQueue&lt;Item&gt;"]
    WorkerThread[Worker Thread]
    NodeX[Node X 实例]
    ModuleX[Module X 实例]
    InQ["Input Queues<br/>(id->queue*)"]
    OutQ["Output Queues<br/>(id->queue*)"]
    OSMX[OutputStreamManager X]
    MirrorsX0[Mirror List X.0]
    MirrorsX1[Mirror List X.1]
    ISM_NextNode["ISM (Next Node)"]

    %% --- 2. 核心组件交互流程 ---
    
    %% 流程 1: Graph I/O
    GI -- "Input" --> ISMX
    GO -- "Output" --> ISMX

    %% 流程 2: 任务创建
    ISMX -- "Creates &amp; Submits" --> TaskInst

    %% 流程 3: 任务调度
    Scheduler -- "Receives & Routes" --> TaskInst
    TaskInst -- "Placed in" --> PQ

    %% 流程 4: 任务执行 (Worker -> Node -> Module)
    WorkerThread -- "Executes" --> TaskInst
    WorkerThread -- "Calls" --> NodeX
    NodeX -- "Calls" --> ModuleX

    %% 流程 5: Module 内部处理
    ModuleX -- "Gets Input From" --> InQ
    ModuleX -- "Puts Output To" --> OutQ
    
    %% 流程 6: 任务后处理与数据传播
    NodeX -- "Calls" --> OSMX
    OSMX -- "Propagates Data via" --> MirrorsX0
    OSMX -- "Propagates Data via" --> MirrorsX1

    %% 流程 7: 链接到下一节点
    MirrorsX0 --> ISM_NextNode
    MirrorsX1 --> ISM_NextNode
```



### **5. 关键流程详解 (图示 + 说明)**

#### **5.1 图的初始化与启动流程**

Code snippet

```mermaid
sequenceDiagram
    participant App as 应用程序
    participant Graph as Graph 实例
    participant Scheduler as Scheduler
    participant Node as Node (代表所有节点)
    participant Module as Module
    participant ISM as InputStreamManager
    participant OSM as OutputStreamManager

    App->>Graph: new Graph(config, ...)
    Graph->>Graph: init(config, ...)
    Graph->>Scheduler: new Scheduler(callbacks, count)
    Graph->>Graph: init_nodes()
    loop 对每个 NodeConfig
        Graph->>Node: new Node(id, node_config, callbacks)
        Node->>Node: ModuleFactory::create_module(...)
        Node->>Module: module->init()
        Node->>ISM: new InputStreamManager(...)
        Node->>OSM: new OutputStreamManager(...)
        Graph->>Graph: nodes_[id] = Node
        opt 是源节点
            Graph->>Graph: source_nodes_.push_back(Node)
        end
    end
    Graph->>Graph: init_input_streams()
    Graph->>Graph: add_all_mirrors_for_output_stream() [建立连接]
    Graph->>Graph: find/delete orphan streams
    loop 对每个 source_node
        Graph->>Scheduler: add_or_remove_node(source_node_id, true)
    end

    App->>Graph: start()
    Graph->>Scheduler: start()
    Scheduler->>Scheduler: 启动所有 SchedulerQueue 线程
    Scheduler->>Scheduler: (可选) 启动 alive_watch 线程
    loop 对每个 orphan_stream
        Graph->>ISM: add_packets(EOF) [向孤立输入推EOF]
    end
    Graph-->>App: 图已启动
```

**流程说明:**

1. **创建 Graph:** 应用程序根据配置创建 `Graph` 对象。
2. **核心 Init:** `Graph` 构造函数调用 `init` 方法。
3. **创建 Scheduler:** `Graph` 创建 `Scheduler` 实例，并设置好回调函数（`get_node_`, `close_report_`）。
4. **创建 Nodes:** `Graph::init_nodes` 遍历配置：
   - 为每个节点创建 `Node` 实例。
   - `Node` 构造函数负责创建 `Module` (通过 `ModuleFactory`) 并调用 `module->init()`，创建 `InputStreamManager` 和 `OutputStreamManager`。
   - 设置 `NodeCallBack` (用于 `Node` 与 `Scheduler` 通信)。
   - 将 `Node` 存入 `Graph::nodes_` map，源节点加入 `source_nodes_` 列表。
5. **创建图输入:** `Graph::init_input_streams` 创建 `GraphInputStream` 实例及其关联的虚拟 `OutputStreamManager`。
6. **建立连接:** `Graph::add_all_mirrors_for_output_stream` 遍历所有 `Node` 的 `OutputStream`，查找下游 `Node` 中标识符匹配的 `InputStream`，并创建 `MirrorStream` 连接。同时，下游 `InputStreamManager` 会记录上游节点 ID (`upstream_nodes_`)。图输入流的虚拟 `OutputStream` 也在此步骤连接到实际的下游节点。
7. **处理孤立流:** 查找并处理未连接的输入/输出流。
8. **启动 Scheduler:** `Graph::start` 调用 `Scheduler::start`，`Scheduler` 启动所有 `SchedulerQueue` 的工作线程和可选的守护线程。
9. **添加源节点:** `Graph` 将所有 `source_nodes_` 加入 `Scheduler` 的待调度列表 (`nodes_to_schedule_`)。
10. **启动完成:** 图现在准备就绪，`Scheduler` 将开始尝试调度源节点。



#### **5.2 单帧数据流转与处理 (Decode -> Scale)**

Code snippet

```mermaid
sequenceDiagram
    participant Scheduler as Scheduler
    participant DecodeNode as Decoder (Node)
    participant DecodeModule as Decoder (Module)
    participant DecodeOSM as Decoder OSM
    participant ScaleISM as Scale ISM
    participant ScaleIS as Scale IS (Queue)
    participant ScaleNode as Scale (Node)
    participant ScaleModule as Scale (Module)
    participant TaskA as Task (for Decoder)
    participant TaskB as Task (for Scale)
    participant SQ as SchedulerQueue

    %% -- Decoder (Source) Task Creation & Execution --
    Scheduler->>DecodeNode: schedule_node() [Triggered initially or periodically]
    DecodeNode->>DecodeNode: is_source() == true
    DecodeNode->>TaskA: new Task(DecodeID, [], [OutID])
    DecodeNode->>TaskA: set_timestamp(...)
    DecodeNode->>Scheduler: callback_.scheduler_cb(TaskA)
    Scheduler->>DecodeNode: inc_pending_task()
    Scheduler->>SQ: add_task(TaskA)
    SQ->>DecodeNode: exec_loop -> process_node(TaskA)
    DecodeNode->>DecodeModule: process(TaskA)
    DecodeModule->>DecodeModule: Decode frame -> Packet pkt_decoded
    DecodeModule->>TaskA: fill_output_packet(OutID, pkt_decoded)

    %% -- Data Propagation --
    DecodeNode->>DecodeOSM: post_process(TaskA)
    DecodeOSM->>DecodeOSM: Get queue from TaskA.outputs_queue_[OutID]
    DecodeOSM->>ScaleISM: add_packets(ScaleInID, copied_queue_with_pkt_decoded)
    ScaleISM->>ScaleIS: add_packets(copied_queue) -> Push pkt_decoded to SafeQueue
    ScaleIS->>Scheduler: callback_.sched_required(ScaleID)

    %% -- Scale Node Scheduling & Task Creation --
    Scheduler->>ScaleNode: schedule_node() [Triggered by sched_required]
    ScaleNode->>ScaleISM: schedule_node()
    ScaleISM->>ScaleISM: get_node_readiness() -> READY
    ScaleISM->>ScaleIS: pop_packet() -> pkt_decoded
    ScaleISM->>TaskB: new Task(ScaleID, [ScaleInID], [ScaleOutID])
    ScaleISM->>TaskB: fill_input_packet(ScaleInID, pkt_decoded)
    ScaleISM->>TaskB: set_timestamp(...)
    ScaleISM->>Scheduler: callback_.scheduler_cb(TaskB)

    %% -- Scale Node Execution --
    Scheduler->>ScaleNode: inc_pending_task()
    Scheduler->>SQ: add_task(TaskB)
    SQ->>ScaleNode: exec_loop -> process_node(TaskB)
    ScaleNode->>ScaleModule: process(TaskB)
    ScaleModule->>TaskB: pop_packet_from_input_queue(ScaleInID) -> pkt_decoded
    ScaleModule->>ScaleModule: Scale pkt_decoded -> pkt_scaled
    ScaleModule->>TaskB: fill_output_packet(ScaleOutID, pkt_scaled)
    ScaleNode->>ScaleNode: post_process(TaskB) ... [Continues to next node]
```

**流程说明:**

1. **源节点 Task 创建:** `Scheduler` 尝试调度 `DecodeNode`。`DecodeNode::schedule_node` 发现自己是源节点，创建 `TaskA` (无输入，有输出)，设置时间戳，通过回调 `scheduler_cb` 提交给 `Scheduler`。
2. **源节点 Task 执行:** `Scheduler` 将 `TaskA` 分配给一个 `SchedulerQueue`。工作线程执行 `DecodeNode::process_node(TaskA)`，进而调用 `DecodeModule::process(TaskA)`。模块解码数据，生成 `Packet pkt_decoded`，放入 `TaskA` 的输出队列 `outputs_queue_`。
3. **数据传播:** `DecodeNode` 处理完成后，调用 `DecodeOSM::post_process(TaskA)`。`DecodeOSM` 从 `TaskA` 的输出队列取出 `pkt_decoded` (通常会拷贝到一个新队列)，找到连接到下游 `ScaleISM` 的 `MirrorStream`，调用 `ScaleISM::add_packets()`。
4. **下游接收:** `ScaleISM::add_packets` 找到对应的 `ScaleIS`，调用 `ScaleIS::add_packets`，将 `pkt_decoded` 放入 `ScaleIS` 的内部 `SafeQueue`。
5. **下游调度请求:** `ScaleIS` 添加数据后，通过 `callback_.sched_required(ScaleID)` 通知 `Scheduler`。
6. **下游调度检查:** `Scheduler` 收到请求，调用 `ScaleNode::schedule_node` (间接通过 `sched_required` -> `to_schedule_queue`)。
7. **下游 Task 创建:** `ScaleNode` 调用 `ScaleISM::schedule_node`。`ScaleISM` 调用 `get_node_readiness` 检查 `ScaleIS` 是否有数据且满足同步策略。如果就绪，创建 `TaskB` (`node_id` 为 Scale 的 ID)，调用 `fill_task_input` 从 `ScaleIS` `pop` 出 `pkt_decoded` 并填充到 `TaskB` 的 `inputs_queue_`，设置时间戳，通过 `scheduler_cb` 提交 `TaskB` 给 `Scheduler`。
8. **下游 Task 执行:** `Scheduler` 将 `TaskB` 分配给 `SchedulerQueue` 执行。工作线程调用 `ScaleNode::process_node(TaskB)`，进而调用 `ScaleModule::process(TaskB)`。模块从 `TaskB` 的输入队列获取 `pkt_decoded`，处理后生成 `pkt_scaled`，放入 `TaskB` 的输出队列... 流程继续向下游传递。



#### **5.3 调度核心流程 (`sched_required`)**

`sched_required` 是驱动调度的关键，由 `InputStreamManager` (数据到达) 或 `Node` (关闭) 调用。

Code snippet

```mermaid
flowchart TD
    A["sched_required(node_id, is_closed) 被调用"] --> B{"is_closed == true ? (节点关闭)"};
    B -- Yes --> C["callback_.close_report_(node_id) 向Graph报告"];
    C --> D["遍历 upstream_nodes_"];
    D --> E["to_schedule_queue(upstream_node)<br/>尝试让上游处理关闭/剩余数据"];
    E --> F[结束];

    B -- No --> G["获取 Node 和 ISM 实例"];
    G --> H["遍历 upstream_nodes_"];
    H --> I["递归调用 sched_required(upstream_id, false)<br/>**向上回溯检查**"];
    I --> H;

    H --> J["std::lock_guard<std::mutex> lk(node->sched_mutex_)"];
    J --> K{"!node->too_many_tasks_pending() &&<br/>!node->any_of_downstream_full() ?<br/>(检查反压)"};
    K -- Yes --> L["node->pre_sched_num_++<br/>(可能用于限制并发调度?)"];
    L --> M["to_schedule_queue(node)<br/>**尝试触发节点调度**"];
    M --> N[结束];
    K -- No --> N;

    style F fill:#eee,stroke:#333,stroke-width:2px;
    style N fill:#eee,stroke:#333,stroke-width:2px;
```

**流程说明:**

1. **判断关闭:** 如果是节点关闭通知，向 Graph 报告，并尝试调度所有上游节点。
2. **回溯上游:** 如果是正常调度请求，先递归调用 `sched_required` 检查所有直接上游节点。**为何回溯?** 可能是为了确保数据依赖的满足（优先让上游产生数据），或者是为了将反压状态向上传递（如果当前节点因为反压不能调度，上游节点在递归检查时也可能因此无法被加入队列）。
3. **检查当前节点:** 锁定节点调度锁 (`sched_mutex_`)。
4. **反压判断:** 检查当前节点是否任务积压 (`too_many_tasks_pending`) 或下游已满 (`any_of_downstream_full`)。
5. **加入调度队列 (尝试):** 如果反压条件不满足，调用 `to_schedule_queue(node)`。
6. **`to_schedule_queue()`**: 调用 `node->schedule_node()`。如果 `schedule_node` 成功创建并提交了 Task，则更新 `last_schedule_clk_` (用于超时检测)。

### **6. 内存管理与零拷贝**

- **核心:** `Packet` 和 `VideoFrame` / `AudioFrame` 内部使用 `std::shared_ptr` 或类似引用计数机制。
- **数据传递:** 当 `Packet` 在队列、`Task`、`Stream` 之间传递时，主要是智能指针的拷贝，实际的媒体数据（可能很大）**不发生拷贝**。
- **`OutputStream::propagate_packets` 中的拷贝:** 创建 `SafeQueue` 副本时，也是智能指针的拷贝，确保每个下游拿到的是指向**同一份**媒体数据的 `Packet`。
- **HMP:** BMF 底层使用 HMP 管理 CPU/GPU 内存，支持更底层的零拷贝。

### **7. 动态图编辑**

- 通过 `Graph::update(GraphConfig)` 实现。
- **`dynamic_add`:** 创建新 `Node`，并通过解析流标识符 (`alias.stream_name`) 动态地修改旧 `Node` 的 `InputStreamManager` 或 `OutputStreamManager` 来添加连接。
- **`dynamic_remove`:** 流程复杂，涉及暂停上下游节点 (`wait_paused`)，注入 `EOF`，等待数据处理完成 (`wait_on_stream_empty`)，断开连接 (`remove_stream`, `remove_upstream_nodes`)，关闭并移除节点，恢复下游节点。
- **`dynamic_reset`:** 调用 `Node::need_opt_reset()` 标记需要重置，`Node::process_node` 中检查标记并调用 `Module::dynamic_reset()`。

### **8. 总结**

BMF 通过精心设计的组件和交互机制，实现了高效、灵活、可扩展的流媒体处理：

- **Graph/Node/Module** 定义了清晰的处理结构。
- **Stream/Manager/Task** 构成了可靠的数据流转和同步机制。
- **Scheduler/Queue** 提供了异步并发执行和反压控制能力。
- **Factory/Manager/Registry** 实现了强大的模块化和插件化。
- **动态图编辑** 赋予了框架极高的灵活性。