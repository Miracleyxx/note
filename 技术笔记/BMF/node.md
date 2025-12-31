## **BMF 学习笔记: Node 组件详解 (`node.h` & `node.cpp`)**

### **1. 设计理念与核心职责**

`Node` 类是 BMF Graph (图) 中的 **基本执行单元**。它像一个“工人”，负责接收数据、执行特定的处理任务，并将结果传递给下一个“工人”。其设计理念和核心职责包括：

1. **封装处理逻辑 (Encapsulation):**
   - `Node` 封装了一个 `Module` 实例。`Module` 包含具体的业务处理算法（如解码、滤镜、编码）。`Node` 负责管理 `Module` 的生命周期 (`init`, `close`) 和调用其处理方法 (`process`)。
   - **为何这样设计?** 将通用的节点管理逻辑 (如流控制、状态管理、调度交互) 与具体的业务逻辑 (在 `Module` 中) 分离，提高了复用性和可维护性。用户可以专注于编写 `Module` 逻辑，而 `Node` 负责将其集成到 BMF 框架中。
2. **流管理中心 (Stream Hub):**
   - `Node` 持有 `InputStreamManager` 和 `OutputStreamManager` 的实例。它负责管理所有进入该节点的数据流（输入流）和从该节点流出的数据流（输出流）。
   - **为何这样设计?** `Node` 作为数据流转的关键节点，需要统一管理其输入和输出。通过 `Managers`，可以实现复杂的流同步策略和数据传播机制。
3. **调度交互点 (Scheduling Interface):**
   - `Node` 是 `Scheduler` (调度器) 进行调度的对象。它需要判断自己是否满足执行条件 (`schedule_node`)，接收并执行 `Scheduler` 分发的 `Task` (`process_node`)，并通过回调 (`NodeCallBack`) 与 `Scheduler` 通信（例如，请求调度 `sched_required`）。
   - **为何这样设计?** 将调度决策 (在 `Scheduler` 中) 与执行逻辑 (在 `Node` 中) 分离。`Node` 只需根据自身状态和数据情况与调度器交互，调度器则负责全局的资源分配和任务排序。
4. **状态维护者 (State Maintainer):**
   - `Node` 维护自身的状态 (`NodeState`: NOT_INITED, RUNNING, PENDING, CLOSED, PAUSE_DONE)，例如是否正在运行、是否已关闭、是否暂停等。
   - 同时，它也追踪一些关键指标，如待处理任务数 (`pending_tasks_`)，用于流量控制。
   - **为何这样设计?** 状态管理对于确保图的正确运行、实现暂停/恢复以及进行资源管理至关重要。
5. **生命周期控制 (Lifecycle Control):**
   - `Node` 具有明确的生命周期，由 `Graph` 创建，并通过 `init`, `close`, `reset` 等方法进行管理。
   - **为何这样设计?** 保证 `Module` 和相关流资源的正确初始化和释放。

### **2. 核心接口详解 (`node.h`)**

**构造与初始化:**

- `Node(int node_id, NodeConfig &node_config, NodeCallBack &callback, std::shared_ptr<Module> pre_allocated_modules, BmfMode mode, std::shared_ptr<ModuleCallbackLayer> callbacks)`: 构造函数，初始化节点的核心组件。

**核心处理流程:**

- `process_node(Task &task)`: **执行任务的核心函数**。接收调度器分发的 `Task`，调用 `Module::process()` 处理数据，处理完成后调用 `OutputStreamManager::post_process()` 传播输出。
- `schedule_node()`: **判断节点是否可被调度的核心函数**。
  - 如果是源节点 (`is_source() == true`)，则自己创建 `Task` 并提交给调度器。
  - 如果是非源节点，则调用 `InputStreamManager::schedule_node()` 来检查输入流是否满足处理条件，如果满足，`InputStreamManager` 会创建 `Task` 并提交。

**状态与生命周期:**

- `close()`: 关闭节点，调用 `Module::close()`，设置状态为 `CLOSED`，并通知调度器。
- `reset()`: 重置节点状态，通常用于 `SERVER_MODE` 下处理完一个请求后复用节点。
- `is_closed()`: 检查节点是否已关闭。
- `set_status(NodeState state)`: 设置节点状态。
- `get_status()`: 获取节点状态字符串。
- `wait_paused()`: 阻塞等待，直到节点处理完当前任务并进入 `PAUSE_DONE` 状态。
- `set_source(bool flag)`: 设置/取消节点的源节点标记。
- `is_source()`: 判断是否为源节点 (没有输入流)。

**任务与流量控制:**

- `inc_pending_task()`: 增加待处理任务计数（当 Task 加入调度队列时）。
- `dec_pending_task()`: 减少待处理任务计数（当 Task 处理完成时）。
- `too_many_tasks_pending()`: 检查待处理任务是否超过阈值 (`max_pending_tasks_`)。用于**反压 (Backpressure)**，防止任务积压过多。
- `any_of_downstream_full()`: 检查是否有任何一个下游节点的输入队列已满 (通过 `OutputStreamManager` 查询)。用于**反压**，防止向下游发送过多数据。
- `any_of_input_queue_full()`: 检查是否有任何一个输入流队列已满。
- `all_input_queue_empty()`: 检查是否所有输入流队列都为空。

**流管理与查询:**

- `get_input_stream_manager(...)`: 获取 `InputStreamManager` 实例。
- `get_output_stream_manager(...)`: 获取 `OutputStreamManager` 实例。
- `get_input_streams(...)`: 获取所有 `InputStream` 实例。
- `get_output_streams(...)`: 获取所有 `OutputStream` 实例。

**动态更新与控制:**

- `need_opt_reset(JsonParam reset_opt)`: 标记节点需要进行动态重置，并传入新的配置选项。`process_node` 会在执行前检查此标记。
- `set_outputstream_updated(bool update)`: 标记节点的输出流配置是否被动态修改过 (例如添加了新的输出流连接)。

**信息获取:**

- `get_id()`: 获取节点 ID。
- `get_alias()`: 获取节点别名 (来自 `NodeConfig`)。
- `get_action()`: 获取节点动态操作类型 (来自 `NodeConfig`, 如 "add", "remove")。
- `get_type()`: 获取模块类型 (如 "c_ffmpeg_decoder")。
- `get_scheduler_queue_id()`: 获取该节点被分配到的调度器队列 ID。
- `set_scheduler_queue_id(...)`: 设置调度器队列 ID。
- `get_schedule_attempt_cnt()`: 获取尝试调度的次数。
- `get_schedule_success_cnt()`: 获取成功调度的次数。
- `get_last_timestamp()`: 获取最后处理的任务的时间戳。
- `get_source_timestamp()`: (仅源节点) 获取下一个递增的时间戳。

**“饥饿检查” (Hungry Check - 优化机制):**

- `is_hungry()`: 检查下游是否“饥饿”（即需要数据）。
- `register_hungry_check_func(...)`: 注册下游节点的饥饿检查函数。
- `get_hungry_check_func(...)`: 获取注册的检查函数。
  - **作用:** 允许源节点在生成数据前检查下游是否真的需要数据，避免不必要的资源消耗。这个检查链通过 `Graph::init` 时建立。

### **3. 核心成员变量详解 (`node.h`)**

- `id_` (int): 节点的唯一 ID。
- `node_config_` (NodeConfig): 该节点的配置信息。
- `callback_` (NodeCallBack): 包含与 Graph 和 Scheduler 通信的回调函数指针。
- `mode_` (BmfMode): 当前 Graph 的运行模式 (NORMAL, SERVER, GENERATOR 等)。
- `module_callbacks_` (shared_ptr<ModuleCallbackLayer>): 模块回调层，用于模块向外部调用函数。
- `module_` (shared_ptr<Module>): 实际执行处理逻辑的模块实例。
- `is_premodule_` (bool): 标记 `module_` 是否是外部预分配传入的。
- `input_stream_manager_` (shared_ptr<InputStreamManager>): 管理所有输入流。
- `output_stream_manager_` (shared_ptr<OutputStreamManager>): 管理所有输出流。
- `state_` (NodeState): 节点的当前状态。
- `is_source_` (bool): 是否为源节点。
- `pending_tasks_` (int): 当前已提交给调度器但尚未完成处理的任务数量。
- `max_pending_tasks_` (int): `pending_tasks_` 的阈值，用于反压控制。
- `queue_size_limit_` (uint32_t): 输入流队列的大小限制 (通常等于 `max_pending_tasks_`)。
- `infinity_node_` (bool): 标记是否为“无限”节点 (如循环模块)，影响关闭逻辑。
- `need_opt_reset_` (bool): 标记是否需要进行动态选项重置。
- `reset_option_` (JsonParam): 存储待应用的动态重置选项。
- `source_timestamp_` (int64_t): (仅源节点) 用于生成递增时间戳。
- `last_timestamp_` (int64_t): 最后处理的任务的时间戳。
- `mutex_` (recursive_mutex): 用于保护节点内部状态的递归互斥锁。
- `sched_mutex_` (mutex): 用于保护与调度相关的特定操作的互斥锁 (如 `sched_required` 中的检查)。
- `pause_mutex_`, `pause_event_`: 用于实现 `wait_paused` 的同步机制。
- `hungry_check_func_`: 存储下游节点的饥饿检查函数。

### **4. 关键实现详解 (`node.cpp`)**

#### **4.1 构造函数 `Node::Node(...)`**

```c++
// node.cpp (构造函数注释)
Node::Node(int node_id, NodeConfig &node_config, NodeCallBack &node_callback,
           std::shared_ptr<Module> pre_allocated_module, BmfMode mode,
           std::shared_ptr<ModuleCallbackLayer> callbacks)
    : id_(node_id), node_config_(node_config), callback_(node_callback),
      mode_(mode), module_callbacks_(callbacks) { // 初始化成员列表

    // 获取模块类型、设置节点名称
    type_ = node_config_.get_module_info().module_name;
    module_name_ = type_;
    node_name_ = "Node_" + std::to_string(id_) + "_" + module_name_;

    // ... (Trace 相关代码) ...

    // 判断是否为源节点 (根据配置中是否有输入流)
    is_source_ = node_config.input_streams.empty();
    // 获取输入队列大小限制
    queue_size_limit_ = node_config_.get_node_meta().get_queue_size_limit();
    // 初始化待处理任务计数
    pending_tasks_ = 0;
    // 设置待处理任务阈值 (等于队列大小)
    max_pending_tasks_ = queue_size_limit_;
    // ... (打印队列大小) ...
    task_processed_cnt_ = 0; // 已处理任务计数
    is_premodule_ = false; // 默认不是预分配模块

    // --- 创建或获取 Module 实例 ---
    if (pre_allocated_module == nullptr) { // 如果没有预分配
        is_premodule_ = false;
        JsonParam node_option_param = node_config_.get_option();
        // 通过 ModuleFactory 创建模块实例
        module_info_ = ModuleFactory::create_module(
            type_, id_, node_option_param, node_config_.module.module_type,
            node_config_.module.module_path, node_config_.module.module_entry,
            module_); // module_ 被赋值为创建的实例

        BMF_TRACE_PROCESS(module_name_.c_str(), "init", START);
        // 调用模块的初始化方法
        module_->init();
        BMF_TRACE_PROCESS(module_name_.c_str(), "init", END);
    } else { // 如果传入了预分配模块
        module_ = pre_allocated_module;
        // 如果是真正的预模块 (来自用户)，调用 reset；否则调用 init
        if (node_config.get_node_meta().get_premodule_id() > 0) {
            is_premodule_ = true;
            module_->reset();
        } else {
            module_->init();
        }
        module_->node_id_ = node_id; // 确保模块知道自己的节点ID
    }

    // 设置模块的回调函数，允许模块调用外部注册的函数
    module_->set_callback([this](int64_t key, CBytes para) -> CBytes {
        return this->module_callbacks_->call(key, para);
    });

    // 检查模块是否为无限节点 (影响关闭逻辑)
    infinity_node_ = module_->is_infinity();

    // --- 创建流管理器 ---
    // 创建输出流管理器
    output_stream_manager_ =
        std::make_shared<OutputStreamManager>(node_config.get_output_streams());

    // 创建输入流管理器的回调结构体
    InputStreamManagerCallBack callback;
    callback.scheduler_cb = callback_.scheduler_cb; // 提交Task给调度器
    callback.throttled_cb = callback_.throttled_cb; // 通知调度器添加/移除节点(反压)
    callback.sched_required = callback_.sched_required; // 通知调度器需要调度
    callback.get_node_cb = callback_.get_node;     // 获取其他Node实例
    callback.notify_cb = [this]() -> bool { return this->schedule_node(); }; // (似乎未使用或用于特殊情况) 内部触发自身调度
    callback.node_is_closed_cb = [this]() -> bool { return this->is_closed(); }; // 检查节点是否关闭

    // 调用工厂函数创建具体类型的 InputStreamManager (如 Immediate, Default, FrameSync 等)
    create_input_stream_manager(node_config.get_input_manager(), node_id,
                                node_config.get_input_streams(),
                                output_stream_manager_->get_stream_id_list(),
                                callback, queue_size_limit_,
                                input_stream_manager_); // input_stream_manager_ 被赋值

    // --- 注册饥饿检查函数 (优化) ---
    for (auto stream_id : input_stream_manager_->stream_id_list_) {
        // 如果模块对某个输入流需要饥饿检查
        if (module_->need_hungry_check(stream_id)) {
            // 将模块的 is_hungry 方法注册到节点的 hungry_check_func_ 映射中
            hungry_check_func_[stream_id].push_back(
                [this, stream_id]() -> bool {
                    return this->module_->is_hungry(stream_id);
                });
        }
    }

    // --- 初始化状态和其他变量 ---
    force_close_ = false; // 是否需要强制关闭 (如果下游都关闭了)
    if (!node_config.get_output_streams().empty()) {
        force_close_ = true; // 有输出流时，默认需要检查下游关闭状态
    }
    state_ = NodeState::RUNNING; // 初始状态为运行
    schedule_node_cnt_ = 0;      // 尝试调度计数
    schedule_node_success_cnt_ = 0; // 成功调度计数
    source_timestamp_ = 0;       // 源节点时间戳计数器
    last_timestamp_ = 0;         // 最后处理的时间戳
    wait_pause_ = false;         // 是否正在等待暂停
    need_opt_reset_ = false;     // 是否需要动态重置
}
```

#### **4.2 `Node::schedule_node()`**

```C++
// node.cpp (schedule_node 注释)
// 判断节点是否应该被调度执行并创建Task
bool Node::schedule_node() {
    // ... (Trace 代码) ...
    // 加锁保护状态检查和Task创建
    std::lock_guard<std::mutex> lock(mutex_); // 使用递归锁保护

    schedule_node_cnt_++; // 增加尝试调度计数

    // 如果节点已关闭或正在等待暂停且有任务在处理中，则不调度
    if (state_ == NodeState::CLOSED || state_ == NodeState::PAUSE_DONE ||
        (wait_pause_ == true && pending_tasks_ != 0)) {
        return false;
    }

    // --- 强制关闭检查 (优化) ---
    // 如果节点配置为需要强制关闭 (通常非 Sink 节点都需要)
    if (need_force_close()) {
        // 检查是否所有直接下游节点都已关闭
        if (all_downstream_nodes_closed()) {
            close(); // 如果下游都关了，自己也关闭
            BMFLOG_NODE(BMF_INFO, id_) << "scheduling failed, all downstream node closed: " << type_;
            return false; // 不再调度
        }
    }

    bool result = false; // 标记是否成功提交了 Task

    // --- 源节点处理 ---
    if (is_source()) {
        // 源节点自己创建 Task
        Task task = Task(id_, // 节点ID
                         input_stream_manager_->stream_id_list_,      // 输入流ID列表 (为空)
                         output_stream_manager_->get_stream_id_list()); // 输出流ID列表

        // 设置时间戳 (无限节点特殊处理，否则递增)
        if (infinity_node_) {
            task.set_timestamp(INF_SRC); // 特殊时间戳
        } else {
            task.set_timestamp(get_source_timestamp()); // 获取并递增时间戳
        }
        // 通过回调将 Task 提交给调度器
        callback_.scheduler_cb(task);
        schedule_node_success_cnt_++; // 增加成功调度计数
        result = true;
    }
    // --- 非源节点处理 ---
    else {
        // (检查输出流是否动态更新过，如果更新过，需要通知ISM更新output_stream_id_list_)
        if (node_output_updated_) { /*...*/ node_output_updated_ = false; }

        // 委托给 InputStreamManager 判断是否满足执行条件并创建 Task
        // InputStreamManager 内部会检查同步策略、获取输入Packet、创建Task并调用 callback_.scheduler_cb
        result = input_stream_manager_->schedule_node();
        if (result)
            schedule_node_success_cnt_++; // 增加成功调度计数
    }
    // (解锁在 lock_guard 析构时自动完成)
    return result; // 返回是否成功提交了 Task
}
```

#### **4.3 `Node::process_node(Task &task)`**

```C++
// node.cpp (process_node 注释)
// 执行调度器分发的 Task
int Node::process_node(Task &task) {
    // 如果节点已关闭或已完成暂停，则直接返回，并减少待处理任务计数
    if (state_ == NodeState::CLOSED || state_ == NodeState::PAUSE_DONE) {
        dec_pending_task();
        return 0;
    }

    // ... (Trace 代码) ...

    // 记录最后处理的时间戳
    last_timestamp_ = task.timestamp();

    // --- 处理特殊时间戳 ---
    // 非源节点收到 BMF_EOF 表示所有输入流结束
    if (not is_source() and task.timestamp() == BMF_EOF) {
        BMFLOG_NODE(BMF_INFO, id_) << "process eof, add node to scheduler";
        // 节点行为类似源节点，可以继续处理内部缓冲或生成结束信号
        set_source(true);
        // (注释掉的代码表明可能曾考虑再次加入调度，但当前逻辑下通常不需要)
    }
    // 收到 EOS 表示流终止 (通常用于 SERVER_MODE)
    else if (task.timestamp() == EOS) {
        dec_pending_task();
        BMFLOG_NODE(BMF_INFO, id_) << "meet eos, close the node and propagate eos pkt";
        // 向所有输出流推送 EOS 信号
        for (auto &iter : task.get_outputs()) {
            iter.second->push(Packet::generate_eos_packet());
        }
        // 调用 post_process 将 EOS 传播下去
        output_stream_manager_->post_process(task);
        close(); // 关闭自身
        return 0; // 不再执行 module_->process
    }

    // --- 准备异常处理 ---
    auto cleanup = [this] { // 定义一个 lambda 用于异常时的清理
        this->task_processed_cnt_++;
        this->dec_pending_task();
        BMFLOG_NODE(BMF_ERROR, this->id_) << "Process node failed, will exit.";
    };

    int result = 0;
    try {
        // --- 处理动态重置选项 ---
        std::lock_guard<std::mutex> lock(opt_reset_mutex_); // 保护重置相关的变量
        if (need_opt_reset_) { // 检查是否有待处理的重置请求
            module_->dynamic_reset(reset_option_); // 调用模块的动态重置方法
            need_opt_reset_ = false; // 清除标记
        }
        // (解锁在 lock_guard 析构时自动完成)

        // --- 调用模块核心处理逻辑 ---
        BMF_TRACE_PROCESS(module_name_.c_str(), "process", START);
        state_ = NodeState::RUNNING; // 设置状态为运行中
        result = module_->process(task); // *** 调用 Module 的 process 方法 ***
        state_ = NodeState::PENDING; // 处理完成，状态改回等待
        BMF_TRACE_PROCESS(module_name_.c_str(), "process", END);

        // 检查模块返回值
        if (result != 0)
            BMF_Error_(BMF_StsBadArg, "[%s] Process result != 0.\n", node_name_.c_str());

    } catch (std::exception &e) { // 捕获标准异常
        BMFLOG_NODE(BMF_ERROR, id_) << "catch exception: " << e.what();
        cleanup(); // 执行清理
        std::rethrow_exception(std::current_exception()); // 重新抛出异常，由调度器捕获
    }

    task_processed_cnt_++; // 增加已处理任务计数

    // --- 处理 DONE 时间戳 (表示输入流真正结束) ---
    if (task.timestamp() == DONE && not is_closed()) {
        BMFLOG_NODE(BMF_INFO, id_) << "Process node end";
        if (mode_ == BmfMode::SERVER_MODE) {
            reset(); // 服务器模式下重置节点以复用
        } else {
            close(); // 其他模式下关闭节点
        }
    }

    // --- 传播输出数据 ---
    // 调用 OutputStreamManager 将 Task 中 outputs_queue_ 的数据包发送到下游
    output_stream_manager_->post_process(task);

    // --- 清理与状态检查 ---
    dec_pending_task(); // 处理完成，减少待处理任务计数

    // 检查是否需要完成暂停操作
    if ((wait_pause_ == true && pending_tasks_ == 0)) {
        state_ = NodeState::PAUSE_DONE; // 设置状态为暂停完成
        pause_event_.notify_all(); // 通知等待在 wait_paused() 的线程
        return 0; // 暂停完成，不再请求调度
    }

    // --- 检查是否阻塞 (主要用于 SERVER_MODE) ---
    bool is_blocked = false;
    if (mode_ == BmfMode::SERVER_MODE && !is_source()) {
        // 检查是否所有输入流都处于阻塞状态 (收到EOF但未收到EOS)
        uint8_t blk_num = 0; /*...*/
        if (blk_num == input_stream_manager_->input_streams_.size())
            is_blocked = true;
    }

    // --- 请求下一次调度 ---
    // 如果节点未阻塞且未关闭，则通知调度器需要再次检查调度
    if (!is_blocked && state_ != NodeState::CLOSED)
        callback_.sched_required(id_, false);

    return 0; // 正常返回
}
```

#### **4.4 其他关键函数**

- **`close()`**: 调用 `module_->close()`，设置状态，通过 `sched_required(id_, true)` 通知调度器（并触发上游检查）。
- **`reset()`**: 调用 `module_->reset()`，清除调度队列中的任务，重置输入流状态。
- **`wait_paused()`**: 使用 `std::condition_variable` (`pause_event_`) 等待 `pending_tasks_` 变为 0 且状态变为 `PAUSE_DONE`。
- **`need_opt_reset()`**: 线程安全地设置动态重置标记和选项。

### **5. 与其他组件的交互总结**

- **`Graph`:** 创建和销毁 `Node`，通过 `NodeCallBack` (间接通过 `SchedulerCallBack`) 接收节点关闭通知。
- **`Scheduler`:** 通过 `NodeCallBack::scheduler_cb` 接收 `Node` (或其 `InputStreamManager`) 提交的 `Task`；调用 `Node::process_node` 执行 `Task`；通过 `NodeCallBack::sched_required` 接收调度请求；通过 `NodeCallBack::throttled_cb` 实现反压。
- **`Module`:** 被 `Node` 创建、初始化、调用 `process` 和 `close`。
- **`InputStreamManager`:** 被 `Node` 创建和持有；负责检查输入流状态，创建 `Task` 并通过 `NodeCallBack::scheduler_cb` 提交；通过 `NodeCallBack::sched_required` 通知 `Scheduler`。
- **~:** 被 `Node` 创建和持有；负责在 `Node::process_node` 后调用 `post_process` 将 `Task` 中的输出 `Packet` 传播给下游。
- **Task:** 由 `Node` (源节点) 或 `InputStreamManager` (非源节点) 创建；传递给 `Scheduler`；由 `Scheduler` 分发给 `Node::process_node`；`Node` 调用 `Module::process(task)`；`Node` 调用 `OutputStreamManager::post_process(task)`。