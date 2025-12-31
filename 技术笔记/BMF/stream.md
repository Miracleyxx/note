## **BMF 学习笔记: Stream 流管理详解**

Stream（流）是 BMF 中连接 `Node`（节点）的数据通道。`InputStreamManager` 和 `OutputStreamManager` 及其包含的 `InputStream` 和 `OutputStream` 类共同构成了 BMF 数据流转和同步的核心机制。

### **1. `InputStream` (`input_stream.h`, `input_stream.cpp`)**

#### **1.1 设计理念与核心职责**

`InputStream` 代表了一个 `Node` 的**单个输入流**。它的核心职责是作为一个**线程安全的缓冲区**，接收来自上游节点的数据包 (`Packet`)，并根据下游 `Node`（通过 `InputStreamManager`）的请求提供这些数据包。

1. **缓冲 (Buffering):**
   - 为异步的节点处理提供解耦。上游节点可以将数据放入 `InputStream` 的队列，不必等待下游节点立即处理。
   - **为何这样设计?** 允许上下游节点以不同的速率工作，提高了系统的整体吞吐量和鲁棒性。
2. **线程安全 (Thread Safety):**
   - 内部使用 `SafeQueue<Packet>` 来存储数据包，确保多个线程（上游写入、下游读取）可以安全地并发访问。
   - **为何这样设计?** BMF 是一个多线程框架，`Scheduler` 可能在不同线程中调度节点，必须保证数据在流转过程中的线程安全。
3. **流量控制 (Flow Control):**
   - 具有 `max_queue_size_` 限制，通过 `is_full()` 方法告知上游是否还能接收数据。这是实现**反压 (Backpressure)** 的基础。
   - **为何这样设计?** 防止数据产生速度远超消费速度时，导致内存无限增长和系统崩溃。
4. **状态管理 (State Management):**
   - 维护流的连接状态 (`connected_`) 和阻塞状态 (`block_`，主要用于 SERVER_MODE)。
   - **为何这样设计?** 允许框架知道流是否有效以及是否应该暂停处理（例如，在等待特定事件时）。
5. **同步与通知 (Synchronization & Notification):**
   - 使用 `std::condition_variable` (`fill_packet_event_`, `stream_ept_`) 在数据到达或队列变空时进行线程间的通知和等待。
   - **为何这样设计?** 实现高效的生产者-消费者模式，避免消费者线程在没有数据时空转 CPU。

#### **1.2 核心接口与成员变量**

```c++
// input_stream.h (关键接口和成员中文注释)
BEGIN_BMF_ENGINE_NS
USE_BMF_SDK_NS

class InputStream {
  public:
    // 构造函数: 初始化流ID, 名称, 队列大小限制, 回调等
    InputStream(int stream_id, std::string const &identifier, /*...*/);
    InputStream(int stream_id, StreamConfig &stream_config, /*...*/);

    // 禁止拷贝构造和赋值
    InputStream(InputStream const &) = delete;
    InputStream &operator=(InputStream const &) = delete;

    /** @brief 向流中添加一批数据包 (通常由 InputStreamManager 调用)
     * @param packets 包含待添加数据包的 SafeQueue 智能指针
     * @return 0 表示成功
     */
    int add_packets(std::shared_ptr<SafeQueue<Packet>> &packets);

    /** @brief 获取指定时间戳或之前的数据包 (用于 DefaultInputManager 等同步策略)
     * 会将队列中 <= timestamp 的 Packet 弹出并返回最后一个满足条件的 Packet
     * @param timestamp 目标时间戳
     * @return Packet 找到的数据包 (可能为空 Packet)
     */
    Packet pop_packet_at_timestamp(int64_t timestamp);

    /** @brief 弹出一个数据包 (用于 ImmediateInputStreamManager 等策略)
     * @param block 是否阻塞等待直到有数据包可用 (默认为 true)
     * @return Packet 弹出的数据包 (如果非阻塞且队列为空，可能为空 Packet)
     */
    Packet pop_next_packet(bool block = true);

    /** @brief 检查队列是否为空
     * @return bool 是否为空
     */
    bool is_empty();

    /** @brief 获取流的ID
     * @return int 流ID
     */
    int get_id();

    /** @brief 检查队列是否已满 (达到 max_queue_size_)
     * @return bool 是否已满
     */
    bool is_full();

    /** @brief 获取流中下一个数据包的时间戳边界 (用于 DefaultInputManager)
     * 如果队列为空，返回 next_time_bounding_ (通常是最后一个包的时间戳+1 或 DONE)
     * 如果队列非空，返回队首包的时间戳
     * @param min_timestamp (输出参数) 返回的时间戳
     * @return bool 队列是否为空 (true 为空)
     */
    bool get_min_timestamp(int64_t &min_timestamp);

    // --- 状态管理 ---
    /** @brief 设置流是否已连接到上游
     * @param connected 连接状态
     */
    void set_connected(bool connected);
    bool is_connected(); // 获取连接状态
    bool get_block();   // 获取阻塞状态 (SERVER_MODE)
    void set_block(bool block); // 设置阻塞状态 (SERVER_MODE)
    void probe_eof(bool probed); // (用于动态移除) 标记流正在探测EOF

    // --- 其他 ---
    std::string get_identifier(); // 获取流的标识符 (如 "video.0")
    std::string get_alias();      // 获取流的别名
    std::string get_notify();     // 获取流的通知类型 (如 "video", "audio")
    void clear_queue();          // 清空队列
    void wait_on_empty();        // 阻塞等待直到队列变空

  public: // (部分公开成员，便于Manager访问)
    int max_queue_size_;                    // 最大队列大小
    std::shared_ptr<SafeQueue<Packet>> queue_; // 实际存储数据包的线程安全队列
    std::string identifier_;                // 唯一标识符
    std::string notify_;                    // 类型通知 (video/audio)
    std::string alias_;                     // 别名
    int stream_id_;                         // 在 Node 内的流索引 (从0开始)
    int node_id_;                           // 所属 Node 的 ID
    // std::string stream_manager_name_;    // (似乎未使用)
    int64_t next_time_bounding_;            // 下一个时间边界 (用于同步)
    // int64_t pop_number_ = 0;             // (似乎未使用) 弹出计数
    mutable std::mutex mutex_;              // 通用互斥锁 (保护非队列操作)
    std::condition_variable fill_packet_event_; // 条件变量，用于在添加数据包时唤醒等待者
    std::mutex stream_m_;                   // 用于 wait_on_empty 的锁
    std::condition_variable stream_ept_;    // 用于 wait_on_empty 的条件变量
    bool block_ = false;                    // 阻塞标记 (SERVER_MODE)
    std::function<void(int, bool)> throttled_cb_; // 节流回调 (通知 Scheduler)
    bool connected_ = false;                // 是否已连接上游
    bool probed_ = false;                   // 是否正在探测EOF (动态移除)
};
```

#### **1.3 关键实现详解 (`input_stream.cpp`)**

- **`add_packets()`**:
  - 遍历传入的 `packets` 队列。
  - 将每个 `Packet` `push` 到内部的 `queue_`。
  - 更新 `next_time_bounding_` 为 `pkt.timestamp() + 1`。
  - **处理 EOF/EOS**: 如果收到 `EOS` 或 `BMF_EOF` 包，将 `next_time_bounding_` 设置为 `DONE`，表示此流数据结束。
  - 调用 `fill_packet_event_.notify_all()` 唤醒可能正在 `pop_next_packet(true)` 中等待的线程。
- **`pop_next_packet(bool block)`**:
  - 尝试从 `queue_` 中 `pop` 一个包。
  - 如果 `pop` 成功，检查是否为 `EOS` 或 `BMF_EOF`，并记录日志。
  - 如果 `pop` 失败 (队列为空)：
    - 通知 `stream_ept_` (可能用于 `wait_on_empty`)。
    - 如果 `block` 为 `true`，则使用 `fill_packet_event_.wait_for` 循环等待，直到队列非空，然后再次尝试 `pop`。
    - 如果 `block` 为 `false`，则直接返回空 `Packet`。
- **`is_full()`**: 返回 `queue_->size() >= max_queue_size_`。
- **`get_min_timestamp()`**: 如果队列为空，返回 `next_time_bounding_`；否则，查看队首 `Packet` (`queue_->front(pkt)`) 并返回其时间戳。

#### **1.4 与其他组件的交互**

- **`InputStreamManager`**: 创建 `InputStream`；调用 `add_packets` 写入数据；调用 `pop_next_packet`/`pop_packet_at_timestamp`/`get_min_timestamp`/`is_empty` 读取数据或状态；调用 `set_block`/`probe_eof` 等管理状态。
- **`Scheduler`**: 间接交互。`add_packets` 后，`InputStreamManager` 会调用 `callback_.sched_required` 通知 `Scheduler`。`InputStream` 内部可能调用 `throttled_cb_` (也是 `Scheduler` 的方法) 来进行反压控制（尽管在 `add_packets` 中未直接体现）。
- **`SafeQueue`**: `InputStream` 内部使用 `SafeQueue` 来保证数据存储的线程安全。

### **2. `InputStreamManager` (`input_stream_manager.h`, `input_stream_manager.cpp`)**

#### **2.1 设计理念与核心职责**

`InputStreamManager` 负责管理一个 `Node` 的**所有输入流 (`InputStream`)**。它的核心职责是**实现流同步策略**，决定 `Node` 何时准备好处理数据，并**创建 `Task` 对象**将准备好的数据包传递给 `Scheduler`。

1. **流聚合与管理 (Aggregation & Management):**
   - 持有一个 `input_streams_` (map)，管理该 `Node` 需要的所有 `InputStream` 实例。
   - **为何这样设计?** 将对多个输入流的管理逻辑集中处理，`Node` 本身无需关心具体的流操作细节。
2. **同步策略实现 (Synchronization Strategy):**
   - 这是一个**抽象基类**，定义了 `get_node_readiness()` 和 `fill_task_input()` 两个**纯虚函数**。具体的同步逻辑（如立即处理、等待所有流、按时间戳对齐）由其子类 (`ImmediateInputStreamManager`, `DefaultInputManager`, `FrameSyncInputStreamManager` 等) 实现。
   - **为何这样设计?** 应用了**策略模式**。使得不同的节点可以根据需求选择不同的输入同步方式，增加了框架的灵活性。添加新的同步策略只需派生新的子类。
3. **任务创建 (Task Creation):**
   - 当 `get_node_readiness()` 判断节点可以处理数据时，`schedule_node()` 方法会：
     - 创建一个 `Task` 对象，传入当前 `node_id_` 以及所有输入和输出流的 ID 列表。
     - 设置 `Task` 的时间戳 (`min_timestamp`)。
     - 调用 `fill_task_input(task)`，由具体的子类负责从各自管理的 `InputStream` 中取出合适的 `Packet` 填充到 `Task` 的 `inputs_queue_` 中。
   - **为何这样设计?** `InputStreamManager` 是最清楚何时以及哪些输入数据已经准备好的组件，因此由它负责创建 `Task` 是最合理的。
4. **调度触发 (Scheduling Trigger):**
   - 当上游节点通过 `add_packets()` 推送数据到其管理的 `InputStream` 时，`InputStreamManager` 会调用 `callback_.sched_required()` 通知 `Scheduler` 当前节点可能有数据需要处理。
   - **为何这样设计?** 实现事件驱动的调度。只有当数据到达时，才需要考虑调度该节点，避免了不必要的轮询。
5. **上游追踪 (Upstream Tracking):**
   - 维护 `upstream_nodes_` 集合，记录所有直接连接到此管理器的上游节点的 ID。
   - **为何这样设计?** 主要用于 `Scheduler::sched_required` 的**回溯**机制，例如在节点关闭时通知所有上游节点，或者在检查调度条件时递归检查上游状态。

#### **2.2 核心接口与成员变量**

```c++
// input_stream_manager.h (关键接口和成员中文注释)
BEGIN_BMF_ENGINE_NS
USE_BMF_SDK_NS

// 前向声明 Node 类
class Node;

// 节点就绪状态枚举
enum class NodeReadiness {
    NOT_READY = 1,         // 未就绪 (数据不足或同步条件未满足)
    READY_FOR_PROCESS = 2, // 已就绪，可以处理数据
    READY_FOR_CLOSE = 3    // 所有输入流都已结束 (收到EOF/EOS)，可以关闭
};

// 回调函数集合结构体
class InputStreamManagerCallBack {
  public:
    std::function<void(Task &)> scheduler_cb = NULL;      // 回调：将创建的 Task 提交给 Scheduler
    // std::function<bool()> notify_cb = NULL;             // (似乎未使用) 通知回调
    std::function<void(int, bool)> throttled_cb = NULL;   // 回调：通知 Scheduler 节流/反压 (与 sched_required 功能重叠?)
    std::function<void(int, bool)> sched_required = NULL; // 回调：通知 Scheduler 需要检查调度
    std::function<bool()> node_is_closed_cb = NULL;       // 回调：检查所属 Node 是否已关闭
    std::function<int(int, std::shared_ptr<Node> &)> get_node_cb = NULL; // 回调：获取 Node 实例
};

// 输入流管理器抽象基类
class InputStreamManager {
  public:
    // 构造函数: 初始化节点ID, 输入流列表, 输出流ID列表, 队列大小, 回调
    InputStreamManager(int node_id, std::vector<StreamConfig> &input_streams,
                       std::vector<int> &output_stream_id_list,
                       uint32_t max_queue_size,
                       InputStreamManagerCallBack &callback);
    virtual ~InputStreamManager() = default; // 虚析构

    // --- 纯虚函数 (子类必须实现) ---
    /** @brief 获取管理器的类型名称 (用于标识策略)
     * @return std::string 类型名 (如 "Immediate", "Default")
     */
    virtual std::string type() = 0;
    /** @brief 检查节点是否准备好处理数据
     * 根据具体的同步策略判断。
     * @param min_timestamp (输出参数) 如果准备好，返回应该处理的时间戳
     * @return NodeReadiness 节点的就绪状态
     */
    virtual NodeReadiness get_node_readiness(int64_t &min_timestamp) = 0;
    /** @brief 从管理的 InputStream 中取出数据包填充到 Task 中
     * @param task 需要被填充的 Task 对象
     * @return bool 是否成功填充了至少一个 Packet
     */
    virtual bool fill_task_input(Task &task) = 0;

    // --- 公共接口 ---
    /** @brief 根据流ID获取 InputStream 实例
     * @param stream_id 流ID
     * @param input_stream (输出参数) 获取到的 InputStream 智能指针
     * @return bool 是否成功找到
     */
    bool get_stream(int stream_id, std::shared_ptr<InputStream> &input_stream);

    /** @brief (动态图) 添加一个新的输入流
     * @param name 新流的标识符
     * @param id (似乎未使用，内部生成?)
     * @return int 新分配的 stream_id
     */
    int add_stream(std::string name, int id); // 'id' parameter seems unused internally

    /** @brief (动态图) 移除一个输入流
     * @param stream_id 要移除的流ID
     * @return int 0 表示成功
     */
    int remove_stream(int stream_id);

    /** @brief 阻塞等待指定的输入流变空
     * @param stream_id 流ID
     * @return int 0 表示成功
     */
    int wait_on_stream_empty(int stream_id);

    /** @brief (较少用) 直接从指定的输入流弹出一个 Packet
     * 通常应通过 schedule_node -> fill_task_input 流程获取 Packet
     * @param stream_id 流ID
     * @param block 是否阻塞等待
     * @return Packet 弹出的数据包
     */
    Packet pop_next_packet(int stream_id, bool block = true);

    /** @brief 尝试调度当前节点
     * 调用 get_node_readiness() 判断是否就绪。如果就绪，则创建 Task,
     * 调用 fill_task_input(), 并通过 scheduler_cb 回调将 Task 提交。
     * @return bool 是否成功提交了 Task
     */
    bool schedule_node();

    /** @brief 向指定的输入流添加一批数据包 (通常由上游 OutputStream 调用)
     * @param stream_id 目标流ID
     * @param packets 包含数据包的 SafeQueue 智能指针
     */
    void add_packets(int stream_id, std::shared_ptr<SafeQueue<Packet>> packets);

    /** @brief 添加一个上游节点的 ID 到记录中
     * @param node_id 上游节点 ID
     * @return int 0 表示成功
     */
    int add_upstream_nodes(int node_id);
    /** @brief 移除一个上游节点的 ID
     * @param node_id 上游节点 ID
     */
    void remove_upstream_nodes(int node_id);
    /** @brief 检查指定的 ID 是否是上游节点
     * @param node_id 要检查的节点 ID
     * @return bool 是否是上游节点
     */
    bool find_upstream_nodes(int node_id);

  public: // (部分公开成员，便于Node或其他组件访问)
    int node_id_;                                        // 所属 Node 的 ID
    std::map<int, std::shared_ptr<InputStream>> input_streams_; // 管理的 InputStream 映射 (ID -> Instance)
    InputStreamManagerCallBack callback_;               // 回调函数集合
    std::vector<std::string> input_stream_names_;       // 输入流名称列表 (按ID顺序)
    std::vector<int> stream_id_list_;                   // 输入流ID列表 (通常是 0, 1, 2...)
    std::vector<int> output_stream_id_list_;            // 所属 Node 的输出流ID列表 (创建Task时需要)
    // mutable std::mutex stream_mutex_;               // (似乎未使用) 流操作锁
    std::map<int, int> stream_done_;                    // 记录流是否结束 (收到EOF/EOS) (ID -> 状态)
    int max_id_;                                        // 用于动态添加流时分配ID
    std::mutex mtx_;                                    // 通用互斥锁 (保护内部状态)
    std::set<int> upstream_nodes_;                      // 上游节点 ID 集合
};

// --- 具体策略子类 ---
// (DefaultInputManager, ImmediateInputStreamManager, ServerInputStreamManager,
//  FrameSyncInputStreamManager, ClockBasedSyncInputStreamManager)
// 它们继承自 InputStreamManager 并实现了纯虚函数 get_node_readiness 和 fill_task_input,
// 各自包含实现特定同步逻辑所需的额外成员变量。

// --- 工厂函数 ---
// 根据类型名称创建具体的 InputStreamManager 实例
int create_input_stream_manager(
    std::string const &manager_type, int node_id,
    std::vector<StreamConfig> input_streams,
    std::vector<int> output_stream_id_list, InputStreamManagerCallBack callback,
    uint32_t queue_size_limit,
    std::shared_ptr<InputStreamManager> &input_stream_manager); // 输出参数
```

#### **2.3 关键实现详解 (`input_stream_manager.cpp`)**

- **构造函数**: 创建所有 `InputStream` 实例并存入 `input_streams_` map。
- **`add_packets()`**:
  - 检查所属 `Node` 是否已关闭。
  - 找到对应的 `InputStream`。
  - 调用 `input_stream->add_packets()` 将数据包放入队列。
  - **调用 `callback_.sched_required(node_id_, false)` 通知调度器**。这是触发下游节点调度的关键步骤。
- **`schedule_node()`**:
  - 调用虚函数 `get_node_readiness()` 获取当前就绪状态和目标时间戳 `min_timestamp`。
  - 如果状态是 `READY_FOR_PROCESS`:
    - 创建 `Task` 对象 (`Task(node_id_, stream_id_list_, output_stream_id_list_)`)。
    - 设置 `task.set_timestamp(min_timestamp)`。
    - 调用虚函数 `fill_task_input(task)` 让子类填充数据。
    - 如果填充成功，调用 `callback_.scheduler_cb(task)` 将 `Task` 提交给 `Scheduler`。
    - 返回 `true`。
  - 如果状态是 `READY_FOR_CLOSE` 或 `NOT_READY`，返回 `false`。
- **`add_upstream_nodes()` / `remove_upstream_nodes()` / `find_upstream_nodes()`**: 简单地对 `upstream_nodes_` 集合进行增、删、查操作。
- **各子类的 `get_node_readiness()` 和 `fill_task_input()`**:
  - **`Immediate`**: `get_node_readiness` 检查是否有**任何** `InputStream` 非空；`fill_task_input` 从所有非空 `InputStream` 中取出**所有**可用的 `Packet` 填充到 `Task`。
  - **`Default`**: `get_node_readiness` 检查**所有** `InputStream` 的 `min_timestamp`，取其中的最小值，并确保没有 `InputStream` 的边界小于这个最小值（即所有流都至少有达到该时间戳的数据或已结束）；`fill_task_input` 则调用 `input_stream->pop_packet_at_timestamp(task.timestamp())` 获取精确时间戳的数据。
  - **`Server`**: 类似 `Immediate`，但也处理 `block_` 状态和 `EOF` 计数 (`stream_eof_`)。
  - **`FrameSync`**: 逻辑最复杂，需要缓存 (`next_pkt_`, `curr_pkt_`)，检查所有流是否都有相同时间戳的帧 (`frames_ready_`)，然后才认为就绪；`fill_task_input` 从内部就绪队列 (`pkt_ready_`) 取出同步好的帧。
  - **`ClockSync`**: 依赖 `stream_id = 0` 的流作为时钟，缓存其他流的数据 (`cache_`)，根据时钟时间戳计算目标PTS，从缓存中找到最接近的 `Packet` (`last_pkt_`) 填充到 `Task`。

#### **2.4 与其他组件的交互**

- **`Node`**: 创建和持有 `InputStreamManager`；其 `schedule_node` 方法会调用 `InputStreamManager::schedule_node`。
- **`InputStream`**: 被 `InputStreamManager` 创建、持有和管理；`InputStreamManager` 调用其方法读写数据和状态。
- **`Scheduler`**: `InputStreamManager` 通过 `callback_.sched_required` 请求调度；通过 `callback_.scheduler_cb` 提交创建好的 `Task`。
- **`Task`**: 由 `InputStreamManager` 在 `schedule_node` 中创建并填充数据。
- **`OutputStream` (上游)**: 通过 `MirrorStream` 调用 `InputStreamManager::add_packets` 将数据写入。

### **3. `OutputStream` (`output_stream.h`, `output_stream.cpp`)**

#### **3.1 设计理念与核心职责**

`OutputStream` 代表了一个 `Node` 的**单个输出流**。它的核心职责是将处理完成的数据包 (`Packet`) **可靠地、可能一对多地分发**给所有连接的下游节点的 `InputStreamManager`。

1. **下游连接管理 (Downstream Connection):**
   - 维护一个 `mirror_streams_` 列表，记录所有连接到此输出流的下游 `InputStreamManager` 及其对应的输入流 ID。
   - **为何这样设计?** 实现 Graph 中节点间的连接。一个节点的输出可以连接到零个、一个或多个下游节点的输入。
2. **数据包传播 (Packet Propagation):**
   - 提供 `propagate_packets()` 方法，负责将一批 `Packet` 发送给所有连接的下游。
   - **为何这样设计?** 这是数据在图中向下游流动的核心机制。
3. **数据复制 (Data Copying):**
   - 在 `propagate_packets` 中，为每个下游 `MirrorStream` 创建一个数据队列的**副本** (`make_shared<SafeQueue<Packet>>(*packets.get())`)。
   - **为何这样设计?** 确保每个下游都能独立地接收和处理所有的数据包。如果直接传递同一个队列，下游的消费会互相影响。虽然创建副本有开销，但由于 `Packet` 内部是共享指针，实际复制的主要是队列结构和 `Packet` 的智能指针，而不是媒体数据本身。

#### **3.2 核心接口与成员变量**

```c++
// output_stream.h (关键接口和成员中文注释)
BEGIN_BMF_ENGINE_NS
USE_BMF_SDK_NS

// 前向声明 InputStreamManager
class InputStreamManager;

// 镜像流结构体: 代表一个到下游输入流的连接
class MirrorStream {
  public:
    // 构造函数
    MirrorStream(std::shared_ptr<InputStreamManager> input_stream_manager, int stream_id);

    std::shared_ptr<InputStreamManager> input_stream_manager_; // 指向目标下游节点的 InputStreamManager
    int stream_id_;                                           // 目标下游 InputStreamManager 中的输入流 ID
};

// 输出流类
class OutputStream {
  public:
    // 构造函数: 初始化流ID, 标识符, 别名, 通知类型
    OutputStream(int stream_id, std::string const &identifier,
                 std::string const &alias = "", std::string const &notify = "");

    /** @brief 添加一个下游连接 (MirrorStream)
     * 在图初始化建立连接时调用。
     * @param input_stream_manager 下游节点的 InputStreamManager
     * @param stream_id 下游 InputStreamManager 中的输入流 ID
     * @return 0 表示成功
     */
    int add_mirror_stream(std::shared_ptr<InputStreamManager> input_stream_manager, int stream_id);

    /** @brief 将一批数据包传播给所有连接的下游
     * 通常由 OutputStreamManager::post_process 调用。
     * @param packets 包含待传播数据包的 SafeQueue 智能指针
     * @return 0 表示成功
     */
    int propagate_packets(std::shared_ptr<SafeQueue<Packet>> packets);

    /** @brief 将当前节点的 ID 添加到所有下游 InputStreamManager 的 upstream_nodes_ 集合中
     * @param node_id 当前节点 (上游节点) 的 ID
     * @return 0 表示成功
     */
    int add_upstream_nodes(int node_id);

    int stream_id_;                      // 输出流在其 Node 内的 ID
    std::string identifier_;             // 唯一标识符 (如 "video.0")
    std::string notify_;                 // 类型通知 (如 "video", "audio")
    std::string alias_;                  // 别名
    std::vector<MirrorStream> mirror_streams_; // 下游连接列表
    // std::set<int> upstream_nodes_; // 注意：upstream_nodes_ 是在下游 ISM 中记录的，这里似乎不需要
};
END_BMF_ENGINE_NS
```

#### **3.3 关键实现详解 (`output_stream.cpp`)**

- **`add_mirror_stream()`**: 简单地创建一个 `MirrorStream` 对象并添加到 `mirror_streams_` 向量中。
- **`propagate_packets()`**:
  - 遍历 `mirror_streams_` 列表。
  - 对于每个 `MirrorStream s`：
    - `auto copy_queue = std::make_shared<SafeQueue<Packet>>(*packets.get());`: **创建原始队列的浅拷贝**。这会复制队列结构和队列中的 `Packet` 智能指针，但不会复制 `Packet` 指向的实际媒体数据。
    - `copy_queue->set_identifier(identifier_);`: 设置队列标识符（用于调试）。
    - `s.input_stream_manager_->add_packets(s.stream_id_, copy_queue);`: 调用下游 `InputStreamManager` 的 `add_packets` 方法，将复制的队列传递过去。
- **`add_upstream_nodes()`**:
  - 遍历 `mirror_streams_`。
  - 调用每个下游 `s.input_stream_manager_->add_upstream_nodes(node_id)`，将当前节点 ID 记录到下游 ISM 的 `upstream_nodes_` 集合中。

#### **3.4 与其他组件的交互**

- **`OutputStreamManager`**: 创建、持有和管理 `OutputStream`；调用 `propagate_packets`。
- **`InputStreamManager` (下游)**: 被 `OutputStream` 持有引用 (通过 `MirrorStream`)；其 `add_packets` 方法被 `OutputStream::propagate_packets` 调用。
- **`Graph`**: 在 `init_nodes` 和 `add_all_mirrors_for_output_stream` 中间接调用 `add_mirror_stream` 来建立连接。

### **4. `OutputStreamManager` (`output_stream_manager.h`, `output_stream_manager.cpp`)**

#### **4.1 设计理念与核心职责**

`OutputStreamManager` 负责管理一个 `Node` 的**所有输出流 (`OutputStream`)**。它的核心职责是在 `Node` 处理完一个 `Task` 后，**收集 `Task` 中的输出数据包，并将它们分发**给各自对应的 `OutputStream` 进行下游传播。

1. **输出流聚合 (Aggregation):**
   - 持有 `output_streams_` (map)，管理该 `Node` 的所有 `OutputStream` 实例。
   - **为何这样设计?** 将对多个输出流的管理集中化。
2. **任务后处理 (Post Processing):**
   - 提供 `post_process(Task &task)` 方法，这是其核心功能。在 `Node::process_node` 调用 `Module::process` 之后被调用。
   - **为何这样设计?** 将数据从 `Task` 的临时输出队列 `outputs_queue_` 转移到持久的 `OutputStream` 进行传播的逻辑与 `Node` 的核心处理循环分离。
3. **反压检查 (Backpressure Check):**
   - 提供 `any_of_downstream_full()` 方法，用于检查是否有任何一个下游 `InputStream` 已满。
   - **为何这样设计?** 这是实现反压机制的关键环节。`Node` 或 `Scheduler` 在决定是否调度一个节点处理新任务前，会调用此方法检查下游是否能够接收结果，如果下游已满，则当前节点应暂停调度。
4. **动态流管理 (Dynamic Stream Management):**
   - 提供 `add_stream()` 和 `remove_stream()` 方法，支持在运行时添加或移除输出流及其连接。
   - **为何这样设计?** 支持 BMF 的动态图编辑功能。

#### **4.2 核心接口与成员变量**

```c++
// output_stream_manager.h (关键接口和成员中文注释)
BEGIN_BMF_ENGINE_NS
USE_BMF_SDK_NS

class OutputStreamManager {
  public:
    // 构造函数: 根据输出流配置创建 OutputStream 实例
    OutputStreamManager(std::vector<StreamConfig> output_streams);

    /** @brief 根据流ID获取 OutputStream 实例
     * @param stream_id 流ID
     * @param output_stream (输出参数) 获取到的 OutputStream 智能指针
     * @return bool 是否成功找到
     */
    bool get_stream(int stream_id, std::shared_ptr<OutputStream> &output_stream);

    /** @brief (动态图) 添加一个新的输出流
     * @param name 新流的标识符
     * @return int 新分配的 stream_id
     */
    int add_stream(std::string name);

    /** @brief 获取所有输出流的 ID 列表
     * @return std::vector<int> 流ID列表
     */
    std::vector<int> get_stream_id_list();

    /** @brief 任务后处理：将 Task 输出队列中的 Packet 传播到下游
     * 在 Node::process_node 调用 Module::process 之后被调用。
     * @param task 包含输出数据包的 Task 对象
     * @return int 0 表示成功
     */
    int post_process(Task &task);

    /** @brief (较少用) 直接将一批 Packet 传播到指定输出流
     * @param stream_id 目标输出流 ID
     * @param packets 包含数据包的 SafeQueue 智能指针
     * @return int 0 表示成功
     */
    int propagate_packets(int stream_id, std::shared_ptr<SafeQueue<Packet>> packets);

    /** @brief 检查是否有任何一个直接下游节点的输入流已满 (用于反压)
     * @return bool 是否有下游已满
     */
    bool any_of_downstream_full();

    /** @brief (动态图移除时) 探测所有下游输入流，标记 EOF
     * 防止在移除过程中数据继续向下游传递。
     */
    void probe_eof();

    /** @brief (动态图) 移除一个输出流或其某个下游连接
     * @param stream_id 要操作的输出流 ID
     * @param mirror_id 要移除的下游连接的输入流 ID (-1 表示移除整个输出流及其所有连接)
     */
    void remove_stream(int stream_id, int mirror_id);

    /** @brief 阻塞等待指定的输出流所有下游都已处理完数据 (队列变空)
     * @param stream_id 输出流 ID
     */
    void wait_on_stream_empty(int stream_id);

    /** @brief 获取所有直接下游节点的 ID 列表
     * @param nodes_id (输出参数) 存储下游节点 ID 的 vector
     * @return int 0 表示成功
     */
    int get_outlink_nodes_id(std::vector<int> &nodes_id);

    // --- 公开成员 ---
    std::map<int, std::shared_ptr<OutputStream>> output_streams_; // 管理的 OutputStream 映射 (ID -> Instance)
    std::vector<int> stream_id_list_;                           // 输出流ID列表 (通常是 0, 1, 2...)

  private:
    int max_id_; // 用于动态添加流时分配ID
};

END_BMF_ENGINE_NS
```

#### **4.3 关键实现详解 (`output_stream_manager.cpp`)**

- **构造函数**:
  - 创建 `OutputStream` 实例并存入 `output_streams_` map。
  - **特殊处理**: 有一段逻辑尝试将名为 "video" 和 "audio" 的流（通过 `notify_` 字段判断）调整到 `stream_id` 为 0 和 1 的位置。这可能是为了某些模块（如图形渲染或音频混合）依赖固定的流索引，是一个需要注意的**约定**。
- **`post_process(Task &task)`**:
  - 遍历 `task.outputs_queue_` (这是一个 `map<stream_id, queue<Packet>>`)。
  - `auto q = std::make_shared<SafeQueue<Packet>>(t.second);`: **创建一个新的 SafeQueue 并拷贝 Task 输出队列中的 Packet**。这样做是因为 `Task` 对象可能是临时的，而数据传播需要异步进行，需要将 Packet 转移到持久的队列中。同样，这里主要是智能指针的拷贝。
  - `output_streams_[t.first]->propagate_packets(q);`: 调用对应 `OutputStream` 的 `propagate_packets` 方法，将拷贝后的队列传递下去。
- **`any_of_downstream_full()`**:
  - 遍历所有的 `output_streams_`。
  - 遍历每个 `OutputStream` 的 `mirror_streams_`。
  - 获取下游 `InputStreamManager` (`mirror_stream.input_stream_manager_`)。
  - 获取下游 `InputStream` (`input_stream_manager_->get_stream(...)`)。
  - 调用 `downstream->is_full()` 检查下游队列是否已满。
  - 如果**任何一个**下游已满，立即返回 `true`。
  - 如果所有下游都未满，返回 `false`。
- **`add_stream()`**: 创建新的 `OutputStream` 实例，分配 `stream_id`，加入 `output_streams_` 和 `stream_id_list_`。
- **`remove_stream()`**:
  - 根据 `mirror_id` 区分是移除特定连接还是整个输出流。
  - 如果移除特定连接，找到对应的 `MirrorStream`，调用下游 `input_stream_manager_->remove_stream()`，然后从 `mirror_streams_` 列表中移除。
  - 如果移除整个流 (`mirror_id == -1`) 或最后一个连接被移除，则从 `output_streams_` 和 `stream_id_list_` 中移除该流。
- **`wait_on_stream_empty()`**: 遍历指定输出流的所有下游 `MirrorStream`，调用下游 `input_stream_manager_->wait_on_stream_empty()`。
- **`probe_eof()`**: 遍历所有输出流的所有下游 `MirrorStream`，获取下游 `InputStream`，调用 `downstream->probe_eof(true)`。

#### **4.4 与其他组件的交互**

- **`Node`**: 创建和持有 `OutputStreamManager`；在 `process_node` 后调用 `post_process`；可能调用 `any_of_downstream_full` 进行反压检查。
- **`OutputStream`**: 被 `OutputStreamManager` 创建、持有和管理；其 `propagate_packets` 方法被 `post_process` 调用。
- **`Task`**: `post_process` 从 `Task` 的 `outputs_queue_` 读取处理结果。
- **`InputStreamManager` (下游)**: 通过 `OutputStream` 和 `MirrorStream` 与之交互，`any_of_downstream_full` 会检查下游 `InputStream` 的状态。