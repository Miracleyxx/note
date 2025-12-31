## **BMF 学习笔记: Graph 组件详解 (`graph.h` & `graph.cpp`)**

### **1. 设计理念与核心职责**

`Graph` 类是 BMF 引擎的核心，扮演着 **整个多媒体处理流程的中央协调器和管理器** 的角色。其主要设计理念围绕以下几点：

1. **配置驱动 (Configuration Driven):**
   - Graph 的结构（包含哪些节点、节点间的连接方式）和行为（运行模式、调度策略）主要由 `GraphConfig` 对象定义。这使得处理流程可以通过配置文件或代码动态构建，提高了灵活性。
   - **为何这样设计?** 将图的结构与具体的执行逻辑分离，便于用户通过修改配置快速定制和调整处理流程，而无需修改引擎核心代码。
2. **生命周期管理 (Lifecycle Management):**
   - `Graph` 全权负责其内部所有 `Node` (节点)、`Stream` (流) 以及 `Scheduler` (调度器) 的创建、初始化、启动、暂停、恢复、关闭和销毁。
   - **为何这样设计?** 确保资源能够被统一、有序地管理，避免资源泄漏或状态不一致的问题。提供了清晰的控制接口 (`start`, `pause_running`, `resume_running`, `close`)。
3. **抽象与封装 (Abstraction & Encapsulation):**
   - `Graph` 向用户隐藏了底层的调度细节、线程管理、数据同步、内存管理等复杂性。用户只需关注定义处理流程（节点和连接）。
   - `GraphInputStream` 和 `GraphOutputStream` 封装了图与外部的数据交互接口，使得向图输入数据和从图获取结果变得简单。
   - **为何这样设计?** 降低用户使用门槛，让用户可以专注于业务逻辑本身，而不是底层的并发和数据流控制。
4. **动态性与灵活性 (Dynamism & Flexibility):**
   - 通过 `update()` 方法，支持在 Graph 运行时动态地添加、移除或重置节点及其配置。
   - **为何这样设计?** 满足需要根据运行时条件调整处理流程的复杂场景，例如动态拉流、切换效果、调整参数等，这是许多高级媒体应用（如导播台）的必需功能。
5. **解耦与通信 (Decoupling & Communication):**
   - `Graph` 通过回调函数 (`SchedulerCallBack`, `NodeCallBack`) 与 `Scheduler` 和 `Node` 进行通信，实现了组件间的解耦。例如，`Node` 不需要直接知道 `Scheduler` 的具体实现，只需通过回调通知调度需求。
   - **为何这样设计?** 提高了系统的模块化程度，便于独立测试、维护和扩展各个组件。
6. **统一入口与状态监控 (Unified Entry & Status Monitoring):**
   - `Graph` 是用户启动和控制处理流程的主要入口。
   - 提供 `status()` 方法，允许外部获取 Graph 的当前运行状态和性能信息。
   - **为何这样设计?** 提供清晰的控制点和可观测性，便于集成到更大的系统中进行管理和监控。

### **2. 核心接口详解 (`graph.h`)**

#### **2.1 `GraphInputStream` 类**

- **职责:** 作为 Graph 的数据输入通道的适配器。外部数据通过它注入到 Graph 内部的某个 `OutputStreamManager`，进而流向下游节点。
- **接口:**
  - `set_manager(std::shared_ptr<OutputStreamManager> &manager)`: 关联一个内部的 `OutputStreamManager`。Graph 输入流本质上扮演了一个虚拟的上游节点输出的角色。
  - `add_packet(Packet &packet)`: 向该输入流添加一个数据包，触发数据流入 Graph。
- **成员:**
  - `manager_`: 指向内部关联的 `OutputStreamManager`。

#### **2.2 `GraphOutputStream` 类**

- **职责:** 作为 Graph 的数据输出通道的适配器。外部通过它可以从 Graph 内部的某个 `InputStreamManager` 拉取处理结果。
- **接口:**
  - `set_manager(std::shared_ptr<InputStreamManager> &input_manager)`: 关联一个内部的 `InputStreamManager`。Graph 输出流本质上扮演了一个虚拟的下游节点输入角色。
  - `poll_packet(Packet &packet, bool block = true)`: 从该输出流拉取一个数据包。`block` 参数决定是否阻塞等待。
  - `set_node_id(int node_id)`: 设置该输出流关联的上游节点 ID（用于调度触发）。
  - `inject_packet(Packet &packet, int index = -1)`: (较少用) 向关联的 `InputStreamManager` 注入一个数据包，通常用于特殊控制或错误处理（如注入EOF）。
- **成员:**
  - `input_manager_`: 指向内部关联的 `InputStreamManager`。
  - `node_id_`: 关联的上游节点的 ID。

#### **2.3 `Graph` 类**

- **职责:** 核心协调器，管理整个处理流程。
- **构造与析构:**
  - `Graph(GraphConfig graph_config, std::map<int, std::shared_ptr<Module>> pre_modules, std::map<int, std::shared_ptr<ModuleCallbackLayer>> callback_bindings)`: 构造函数，进行基础设置（如信号处理、日志），并调用 `init`。允许传入预先分配的 `Module` 实例和回调绑定。
  - `~Graph()`: 析构函数，确保 `Scheduler` 被关闭。
- **初始化:**
  - `init(...)`: 核心初始化逻辑，根据 `GraphConfig` 创建 `Scheduler`、初始化所有 `Node` (`init_nodes`)、初始化图的输入流 (`init_input_streams`)、建立节点间的连接 (`add_all_mirrors_for_output_stream`)、处理未连接的流 (`find_orphan_input_streams`, `delete_orphan_output_streams`)，并将源节点加入调度器。
  - `init_nodes()`: 创建所有 `Node` 实例，设置调度器队列 ID，建立 `NodeCallBack`，创建 `InputStreamManager` 和 `OutputStreamManager`，并建立节点间的 `MirrorStream` 连接。
  - `init_input_streams()`: 根据 `GraphConfig` 创建 `GraphInputStream` 实例及其关联的 `OutputStreamManager`，并将它们连接到相应的下游节点。
  - `add_all_mirrors_for_output_stream(...)`: 遍历所有节点，查找与给定 `OutputStream` 标识符匹配的 `InputStream`，并建立 `MirrorStream` 连接。
  - `find_orphan_input_streams()`: 找出没有上游连接的 `InputStream`。
  - `delete_orphan_output_streams()`: 删除没有下游连接的 `OutputStream`，避免潜在问题。
- **生命周期控制:**
  - `start()`: 启动 `Scheduler`，开始处理流程（调度器会首先调度源节点）。向孤立输入流（`orphan_streams_`）推送 EOF。
  - `pause_running(double_t timeout = -1)`: 暂停 `Scheduler` 的调度。可设置超时自动恢复。
  - `resume_running()`: 恢复 `Scheduler` 的调度。
  - `close()`: 优雅关闭。等待所有节点处理完成（通过 `cond_close_` 条件变量同步），然后关闭 `Scheduler`。如果调度器线程有未捕获异常，则重新抛出。
  - `force_close()`: 强制关闭所有节点和调度器，不等节点完成。
  - `quit_gracefully()`: 信号处理函数调用的接口，用于尝试强制关闭。
- **动态更新:**
  - `update(GraphConfig update_config)`: 根据 `update_config` 中的指令（add, remove, reset）动态修改图。这是实现动态功能的关键，逻辑复杂，需要处理节点暂停、连接重建、EOF 注入、状态同步等。
- **数据 I/O:**
  - `add_input_stream_packet(std::string const &stream_name, Packet &packet, bool block = false)`: 向指定的 `GraphInputStream` 推送数据包。
  - `poll_output_stream_packet(std::string const &stream_name, bool block = true)`: 从指定的 `GraphOutputStream` 拉取数据包。
  - `add_eos_packet(std::string const &stream_name)`: 向指定的 `GraphInputStream` 推送流结束 (EOS) 信号。
- **状态与查询:**
  - `get_node(int node_id, std::shared_ptr<Node> &node)`: 根据 ID 获取 `Node` 实例。
  - `all_nodes_done()`: 检查是否所有节点都已关闭。
  - `status()`: 获取图的当前运行信息 (`GraphRunningInfo`)。
  - `print_node_info_pretty()`: 打印所有节点的简要状态信息。
- **内部辅助:**
  - `get_hungry_check_func(...)`, `get_hungry_check_func_for_sources()`: 用于建立源节点的“饥饿检查”函数链，这是一种优化，允许源节点知道下游是否需要数据，从而避免不必要的计算或数据生成。

### **3. 代码详解与中文注释 (`graph.h` & `graph.cpp`)**

**`graph.h`**

```C++
// graph.h (关键部分注释)
#ifndef BMF_GRAPH_H
#define BMF_GRAPH_H

// ... (Includes) ...

BEGIN_BMF_ENGINE_NS // BMF引擎命名空间开始
USE_BMF_SDK_NS      // 使用BMF SDK命名空间

// 图的状态枚举 (当前注释标记为未使用)
enum class GraphState { /*...*/ };

// 图输入流类: 封装了图的数据入口点
class GraphInputStream {
  public:
    // 设置内部关联的OutputStreamManager (图输入流模拟一个上游节点的输出)
    void set_manager(std::shared_ptr<OutputStreamManager> &manager);
    // 向图中添加数据包
    void add_packet(Packet &packet);
    // 指向内部关联的OutputStreamManager
    std::shared_ptr<OutputStreamManager> manager_;
};

// 图输出流类: 封装了图的数据出口点
class GraphOutputStream {
  public:
    // 设置内部关联的InputStreamManager (图输出流模拟一个下游节点的输入)
    void set_manager(std::shared_ptr<InputStreamManager> &input_manager);
    // 从图中拉取数据包 (block=true表示阻塞等待)
    void poll_packet(Packet &packet, bool block = true);
    // 设置该输出流直接关联的上游节点ID (用于调度触发)
    void set_node_id(int node_id){node_id_ = node_id;};
    // (较少用) 向关联的InputStreamManager注入数据包 (如EOF)
    void inject_packet(Packet &packet, int index = -1);
    // 指向内部关联的InputStreamManager
    std::shared_ptr<InputStreamManager> input_manager_;
    // 关联的上游节点ID
    int node_id_;
};

// Graph核心类: 管理整个处理流程
class Graph {
  public:
    // 构造函数: 接收图配置、预分配模块、回调绑定
    Graph(
        GraphConfig graph_config,
        std::map<int, std::shared_ptr<Module>> pre_modules,
        std::map<int, std::shared_ptr<ModuleCallbackLayer>> callback_bindings);
    // 析构函数: 负责资源清理
    ~Graph();

    // 核心初始化函数 (由构造函数调用)
    void init(/*...*/);

    // 初始化所有节点
    int init_nodes();
    // 初始化图输入流
    int init_input_streams();
    // 为指定的OutputStream建立所有下游连接 (MirrorStream)
    int add_all_mirrors_for_output_stream(std::shared_ptr<OutputStream> &stream);
    // 查找未连接上游的InputStream (孤立输入流)
    int find_orphan_input_streams();
    // 删除未连接下游的OutputStream (孤立输出流)
    int delete_orphan_output_streams();

    // 启动图处理流程 (启动调度器)
    int start();
    // 动态更新图 (添加/删除/重置节点)
    int update(GraphConfig update_config);

    // 检查所有节点是否都已完成/关闭
    bool all_nodes_done();

    // 暂停图的运行 (暂停调度器)
    void pause_running(double_t timeout = -1);
    // 恢复图的运行
    void resume_running();
    // 优雅关闭图 (等待所有节点完成)
    int close();
    // 强制关闭图 (不等节点完成)
    int force_close();

    // 向指定的图输入流添加数据包
    int add_input_stream_packet(/*...*/);
    // 从指定的图输出流拉取数据包
    Packet poll_output_stream_packet(/*...*/);
    // 向指定的图输入流添加EOS (流结束) 包
    int add_eos_packet(/*...*/);

    // 根据ID获取节点实例
    int get_node(int node_id, std::shared_ptr<Node> &node);

    // 优雅退出 (通常由信号处理函数调用)
    void quit_gracefully();
    // 打印节点状态信息
    void print_node_info_pretty();
    // 获取图的当前运行状态
    bmf::GraphRunningInfo status();

  private:
    // ... (私有成员变量) ...
    GraphConfig graph_config_; // 图的配置信息
    std::shared_ptr<Scheduler> scheduler_; // 调度器实例
    std::map<std::string, std::shared_ptr<GraphInputStream>> input_streams_; // 图输入流映射
    std::map<std::string, std::shared_ptr<GraphOutputStream>> output_streams_; // 图输出流映射
    std::map<int, std::shared_ptr<Node>> nodes_; // 所有节点的映射 (ID -> Node)
    std::vector<std::shared_ptr<Node>> source_nodes_; // 源节点列表
    std::vector<std::shared_ptr<InputStream>> orphan_streams_; // 孤立输入流列表
    // ... (用于同步关闭的条件变量和计数器) ...
    std::condition_variable cond_close_;
    std::mutex con_var_mutex_;
    int32_t closed_count_;
    bool exception_from_scheduler_; // 标记是否有来自调度器的异常
};

END_BMF_ENGINE_NS // BMF引擎命名空间结束
#endif // BMF_GRAPH_H
```

**`graph.cpp`**

```C++
// graph.cpp (关键部分注释)
#include "../include/common.h"
#include "../include/graph.h"
// ... (Includes) ...

BEGIN_BMF_ENGINE_NS // BMF引擎命名空间开始
USE_BMF_SDK_NS      // 使用BMF SDK命名空间

// 全局Graph指针列表 (用于信号处理)
std::vector<Graph *> g_ptr;

// SIGTERM信号处理函数 (终止信号)
void terminate(int signum) {
    std::cout << "terminated, ending bmf gracefully..." << std::endl;
    for (auto p : g_ptr)
        p->quit_gracefully(); // 调用Graph的优雅退出
}

// SIGINT信号处理函数 (中断信号, 如Ctrl+C)
void interrupted(int signum) {
    std::cout << "interrupted, ending bmf gracefully..." << std::endl;
    for (auto p : g_ptr)
        p->quit_gracefully(); // 调用Graph的优雅退出
}

// Graph构造函数
Graph::Graph(
    GraphConfig graph_config,
    std::map<int, std::shared_ptr<Module>> pre_modules,
    std::map<int, std::shared_ptr<ModuleCallbackLayer>> callback_bindings) {
    // 注册信号处理函数
    std::signal(SIGTERM, terminate);
    std::signal(SIGINT, interrupted);
    // 配置日志
    configure_bmf_log();
    BMFLOG(BMF_INFO) << "BMF Version: " << BMF_VERSION;
    // ... (打印版本信息) ...
    BMFLOG(BMF_INFO) << "start init graph";
    BMF_TRACE(GRAPH_START, "Init"); // 性能追踪点
    // 调用核心初始化函数
    init(graph_config, pre_modules, callback_bindings);
    // 将自身添加到全局列表，以便信号处理
    g_ptr.push_back(this);
}

// Graph核心初始化函数
void Graph::init(
    GraphConfig graph_config,
    std::map<int, std::shared_ptr<Module>> &pre_modules,
    std::map<int, std::shared_ptr<ModuleCallbackLayer>> &callback_bindings) {
    // 保存配置和预分配资源
    graph_config_ = graph_config;
    pre_modules_ = pre_modules;
    callback_bindings_ = callback_bindings;
    mode_ = graph_config.get_mode(); // 获取运行模式

    // 计算所需的调度器数量 (基于节点配置的最大scheduler ID + 1 或 配置中指定)
    int max_scheduler_index = 0;
    for (auto &node_config : graph_config_.get_nodes()) {
        max_scheduler_index =
            std::max(node_config.get_scheduler(), max_scheduler_index);
    }
    scheduler_count_ = max_scheduler_index + 1;
    if (graph_config.get_option().json_value_.count("scheduler_count"))
        scheduler_count_ = graph_config.get_option() /*...*/;

    // --- 设置调度器回调函数 ---
    SchedulerCallBack scheduler_callback;
    // 回调: 让调度器能根据node_id获取Node实例
    scheduler_callback.get_node_ = [this](int node_id, std::shared_ptr<Node> &node) -> int {
        return this->get_node(node_id, node);
    };
    // 回调: 节点关闭时通知Graph (用于等待所有节点完成)
    scheduler_callback.close_report_ = [this](int node_id, bool is_exception) -> int {
        std::lock_guard<std::mutex> _(this->con_var_mutex_);
        this->closed_count_++; // 关闭计数加一
        if (is_exception) { // 如果是异常关闭
            // ... (日志记录, 标记异常来源) ...
            // 如果有图输出流，注入EOF以通知下游
            if (this->output_streams_.size() > 0) { /*...*/ }
        } else {
            // ... (日志记录正常关闭) ...
        }
        // 如果所有节点都关闭了，或者发生了异常，通知等待在close()处的线程
        if (this->closed_count_ == this->nodes_.size() || is_exception)
            this->cond_close_.notify_one();
        return 0;
    };

    // 获取调度器超时配置 (如果存在)
    double time_out = 0;
    if (graph_config.get_option().json_value_.count("time_out")) { /*...*/ }

    // 创建调度器实例
    scheduler_ = std::make_shared<Scheduler>(scheduler_callback, scheduler_count_, time_out);
    BMFLOG(BMF_INFO) << "scheduler count" << scheduler_count_;

    // --- 初始化图的结构 ---
    // 1. 创建所有节点和它们的输出流管理器及输出流实例
    init_nodes();
    // 2. (优化) 为源节点建立饥饿检查函数链 (暂时忽略细节)
    // get_hungry_check_func_for_sources();
    // 3. 初始化图的输入流及其关联的输出流管理器，并连接到下游节点
    init_input_streams();
    // 4. 查找未连接上游的输入流
    find_orphan_input_streams();
    // 5. 删除未连接下游的输出流 (防止配置错误导致问题)
    delete_orphan_output_streams();

    // 将所有源节点加入调度器的初始待调度列表
    for (auto &node : source_nodes_)
        scheduler_->add_or_remove_node(node->get_id(), true);
}


// 初始化所有节点 (非常核心的函数)
int Graph::init_nodes() {
    // --- 设置节点回调函数 ---
    NodeCallBack callback;
    // 回调: 让节点能根据node_id获取其他Node实例 (虽然Node通常不直接交互)
    callback.get_node = [this](/*...*/) { return this->get_node(/*...*/); };
    // 回调: 节点通知调度器自己是否应该被加入/移出调度列表 (基于反压)
    callback.throttled_cb = [this](/*...*/) { return this->scheduler_->add_or_remove_node(/*...*/); };
    // 回调: 节点通知调度器需要调度 (核心通信机制)
    callback.sched_required = [this](/*...*/) { return this->scheduler_->sched_required(/*...*/); };
    // 回调: 节点(主要是ISM)创建Task后提交给调度器执行
    callback.scheduler_cb = [this](/*...*/) { return this->scheduler_->schedule_node(/*...*/); };
    // 回调: 清理特定节点在调度队列中的任务 (用于关闭或重置)
    callback.clear_cb = [this](/*...*/) { return this->scheduler_->clear_task(/*...*/); };

    // --- 遍历配置，创建节点实例 ---
    for (auto &node_config : graph_config_.get_nodes()) {
        std::shared_ptr<Module> module_pre_allocated;
        auto node_id = node_config.get_id();
        // 检查是否有预分配的模块
        if (pre_modules_.count(node_id) > 0)
            module_pre_allocated = pre_modules_[node_id];
        std::shared_ptr<Node> node;

        // 确保每个节点都有回调绑定层
        if (!callback_bindings_.count(node_id))
            callback_bindings_[node_id] = std::make_shared<ModuleCallbackLayer>();

        // 创建Node实例 (传入ID, 配置, 回调, 预分配模块, 模式, 回调绑定)
        node = std::make_shared<Node>(node_id, node_config, callback,
                                      module_pre_allocated, mode_,
                                      callback_bindings_[node_id]);

        // 设置节点应该在哪个调度器队列执行 (基于配置)
        if (node_config.get_scheduler() < scheduler_count_) {
            node->set_scheduler_queue_id((node_config.get_scheduler()));
            // ... (日志) ...
        } else {
            node->set_scheduler_queue_id(0); // 默认或超限时使用队列0
            // ... (日志) ...
        }
        // 将创建的节点存入Graph的节点映射
        nodes_[node_config.get_id()] = node;

        // 判断是否为源节点 (没有输入流配置)
        if ((node_config.get_input_streams().size()) == 0) {
            source_nodes_.push_back(node);
        }
    }

    // --- 创建节点间的连接 (MirrorStream) ---
    for (auto &node_iter : nodes_) { // 遍历所有节点 (作为上游)
        std::map<int, std::shared_ptr<OutputStream>> output_streams;
        node_iter.second->get_output_streams(output_streams);
        // 遍历该节点的每个输出流
        for (auto &output_stream : output_streams) {
            // 为这个输出流查找并添加所有下游连接
            add_all_mirrors_for_output_stream(output_stream.second);
        }
    }
    // (设置OutputStream的上游节点ID, 用于反向查找)
    for (auto &node_iter : nodes_) { /*...*/ }

    // --- 创建图的输出流 (GraphOutputStream) ---
    for (auto &graph_output_stream_config : graph_config_.output_streams) {
        // 遍历所有节点，查找哪个节点的输出流配置与图输出流配置匹配
        for (auto &node_config : graph_config_.nodes) {
            int stream_idx = 0;
            for (auto &node_output_stream_config : node_config.output_streams) {
                if (node_output_stream_config.get_identifier() ==
                    graph_output_stream_config.get_identifier()) {
                    // 找到了匹配的节点内部输出流

                    // 创建一个虚拟的下游InputStreamManager来接收这个流的数据
                    std::shared_ptr<InputStreamManager> virtual_input_manager;
                    create_input_stream_manager("immediate", -1, {node_output_stream_config}, {},
                                                InputStreamManagerCallBack(), 5,
                                                virtual_input_manager);
                    // 创建GraphOutputStream实例
                    auto g_out_s = std::make_shared<GraphOutputStream>();
                    // 将虚拟的InputStreamManager设置给它
                    g_out_s->set_manager(virtual_input_manager);
                    g_out_s->set_node_id(node_config.get_id()); // 记录上游节点ID

                    // 获取实际的上游节点的OutputStream
                    std::map<int, std::shared_ptr<OutputStream>> actual_output_streams;
                    nodes_[node_config.id]->get_output_streams(actual_output_streams);
                    // 将虚拟的InputStreamManager作为Mirror添加到实际的OutputStream
                    actual_output_streams[stream_idx]->add_mirror_stream(virtual_input_manager, 0);

                    // 将GraphOutputStream实例存入Graph的输出流映射
                    output_streams_[graph_output_stream_config.get_identifier()] = g_out_s;
                }
                ++stream_idx;
            }
        }
    }

    return 0;
}

// 初始化图输入流
int Graph::init_input_streams() {
    for (auto &stream_config : graph_config_.get_input_streams()) {
        // 创建GraphInputStream实例
        std::shared_ptr<GraphInputStream> graph_input_stream = std::make_shared<GraphInputStream>();
        // 创建一个关联的OutputStreamManager (因为GraphInputStream模拟上游输出)
        std::vector<StreamConfig> ss = {stream_config};
        std::shared_ptr<OutputStreamManager> manager = std::make_shared<OutputStreamManager>(ss);
        graph_input_stream->set_manager(manager);

        // 存入Graph的输入流映射
        input_streams_[stream_config.get_identifier()] = graph_input_stream;

        // 获取这个模拟输出流
        std::shared_ptr<OutputStream> output_stream;
        manager->get_stream(0, output_stream);
        // 为这个模拟输出流查找并添加所有实际的下游连接
        add_all_mirrors_for_output_stream(output_stream);
    }
    return 0;
}

// 为指定的OutputStream添加所有下游MirrorStream连接
int Graph::add_all_mirrors_for_output_stream(std::shared_ptr<OutputStream> &output_stream) {
    // 遍历图中所有节点 (作为潜在的下游)
    for (auto &node_iter : nodes_) {
        // 源节点没有输入流管理器，跳过
        if (not node_iter.second->is_source()) {
            std::shared_ptr<InputStreamManager> input_stream_manager;
            node_iter.second->get_input_stream_manager(input_stream_manager);
            // 遍历下游节点的每个输入流
            for (auto &input_stream_pair : input_stream_manager->input_streams_) {
                // 如果下游输入流的标识符与当前输出流匹配
                if (output_stream->identifier_ == input_stream_pair.second->identifier_) {
                    // 添加MirrorStream连接: (下游ISM指针, 下游输入流ID)
                    output_stream->add_mirror_stream(input_stream_manager, input_stream_pair.first);
                    // 标记下游输入流已连接
                    input_stream_pair.second->set_connected(true);
                    // (下游ISM记录上游节点ID, BMF此处逻辑似乎在init_nodes()中处理更清晰)
                    // ... input_stream_manager->add_upstream_nodes(...) ...
                }
            }
        }
    }
    return 0;
}


// --- 动态更新 (update) ---
// update函数非常复杂，因为它需要处理运行时修改图结构的各种情况
int Graph::update(GraphConfig update_config) {
    BMFLOG(BMF_INFO) << "dynamic update start: " /*...*/;
    // ... (调用Optimizer进行配置转换) ...

    // --- 解析更新指令 ---
    JsonParam option = update_config.get_option();
    // ... (获取节点操作列表: add, remove, reset) ...
    std::vector<NodeConfig> nodes_add, nodes_remove, nodes_reset;
    // ... (根据action分类NodeConfig) ...

    // --- 处理添加节点 (nodes_add) ---
    if (nodes_add.size()) {
        std::map<int, std::shared_ptr<Node>> added_nodes; // 存储新添加的节点
        std::vector<std::shared_ptr<Node>> added_src_nodes; // 新添加的源节点
        for (auto &node_config : nodes_add) {
            // ... (创建Node实例，与init_nodes逻辑类似) ...
            // 设置回调、创建模块、设置调度器队列ID等
            // ...
            nodes_[node_config.get_id()] = node; // 加入Graph的节点列表
            added_nodes[node_config.get_id()] = node; // 记录新添加的节点
            if (node->is_source()) { added_src_nodes.push_back(node); }
        }

        // --- 建立新节点间的连接 和 新节点与旧节点的连接 ---
        for (auto &new_node_iter : added_nodes) { // 遍历新节点 (作为上游)
            // ... (获取新节点的输出流) ...
            for (auto &output_stream_pair : output_streams) {
                bool connected_within_added = false;
                // 1. 尝试连接到其他 *新添加* 的节点
                for (auto &other_new_node_iter : added_nodes) {
                     // ... (检查标识符匹配, 添加MirrorStream) ...
                     // ... (更新下游ISM的upstream_nodes_) ...
                     // connected_within_added = true; break;
                }
                // 2. 如果没有连接到新节点，尝试连接到 *旧* 节点
                if (!connected_within_added) {
                    // 解析输出流标识符，获取目标节点别名 (约定格式: "alias.stream_name")
                    std::string target_alias = output_stream.second->identifier_.substr(/*...*/);
                    for (auto &existing_node_pair : nodes_) { // 遍历所有节点(包括旧的)
                        // 跳过新添加的节点，并且节点不是源节点
                        if (added_nodes.count(existing_node_pair.first) == 0 && !existing_node_pair.second->is_source()) {
                            // 如果旧节点的别名与目标别名匹配
                            if (target_alias == existing_node_pair.second->get_alias()) {
                                // 获取旧节点的ISM
                                std::shared_ptr<InputStreamManager> existing_ism;
                                existing_node_pair.second->get_input_stream_manager(existing_ism);
                                // 动态地为旧节点的ISM添加一个新的输入流
                                int new_stream_id = existing_ism->add_stream(output_stream.second->identifier_, existing_node_pair.second->get_id());
                                // 添加MirrorStream连接
                                output_stream.second->add_mirror_stream(existing_ism, new_stream_id);
                                // 更新旧节点ISM的上游节点列表
                                if (!existing_ism->find_upstream_nodes(new_node_iter.second->get_id())) {
                                    existing_ism->add_upstream_nodes(new_node_iter.second->get_id());
                                }
                                // 标记新添加的流已连接
                                existing_ism->input_streams_[new_stream_id]->set_connected(true);
                                // ... (日志) ...
                            }
                        }
                    }
                }
            }
        }
        // --- 处理新节点的上游连接 (连接到旧节点) ---
        for (auto &new_node_iter : added_nodes) { // 遍历新节点 (作为下游)
            // ... (获取新节点的ISM) ...
            for (auto &input_stream_pair : input_stream_manager->input_streams_) {
                // 如果输入流未连接 (说明需要连接到旧节点)
                if (!input_stream_pair.second->is_connected()) {
                    // 解析输入流标识符，获取源节点别名
                    std::string source_alias = input_stream.second->identifier_.substr(/*...*/);
                    for (auto &existing_node_pair : nodes_) { // 遍历所有节点(包括旧的)
                         // 跳过新添加的节点
                        if (added_nodes.count(existing_node_pair.first) == 0) {
                            // 如果旧节点的别名与源别名匹配
                            if (source_alias == existing_node_pair.second->get_alias()) {
                                // 获取旧节点的OSM
                                std::shared_ptr<OutputStreamManager> existing_osm;
                                existing_node_pair.second->get_output_stream_manager(existing_osm);
                                // 动态地为旧节点的OSM添加一个新的输出流
                                int new_stream_id = existing_osm->add_stream(input_stream.second->identifier_);
                                // 为这个新输出流添加MirrorStream连接到新节点的ISM
                                existing_osm->output_streams_[new_stream_id]->add_mirror_stream(input_stream_manager, input_stream_pair.second->get_id());
                                // 标记新节点的输入流已连接
                                input_stream_manager->input_streams_[input_stream_pair.second->get_id()]->set_connected(true);
                                // 标记旧节点的输出流已更新 (可能影响其内部状态)
                                existing_node_pair.second->set_outputstream_updated(true);
                                // 更新新节点ISM的上游节点列表
                                if (!input_stream_manager->find_upstream_nodes(existing_node_pair.second->get_id())) {
                                    input_stream_manager->add_upstream_nodes(existing_node_pair.second->get_id());
                                }
                                // ... (日志) ...
                            }
                        }
                    }
                }
            }
        }

        // 将新添加的源节点加入调度器
        for (auto &node : added_src_nodes)
            scheduler_->add_or_remove_node(node->get_id(), true);
    }

    // --- 处理移除节点 (nodes_remove) ---
    if (nodes_remove.size()) {
        for (auto &node_config : nodes_remove) {
            // 1. 根据别名找到要移除的节点 (rm_node)
            int id_of_rm_node = -1; /*...*/
            std::shared_ptr<Node> rm_node = nodes_[id_of_rm_node];

            // 2. 暂停所有直接上游节点
            std::vector<std::shared_ptr<Node>> paused_upstream_nodes;
            // ... (遍历rm_node的input_streams, 找到上游节点, 调用wait_paused()) ...
            // ... (记录上游节点的哪些OutputStream连接到了rm_node) ...

            // 3. 探测 rm_node 的输出流管理器，标记EOF（防止数据继续向下游传递）
            rm_node->get_output_stream_manager()->probe_eof();

            // 4. 向 rm_node 的所有输入流注入EOF包
            for (auto stream_pair : input_stream_manager->input_streams_) {
                 /*...*/ q->push(Packet::generate_eof_packet()); stream_pair.second->add_packets(q);
            }

            // 5. 等待 rm_node 处理完所有输入 (消费掉EOF)
            for (auto stream_pair : input_stream_manager->input_streams_)
                input_stream_manager->wait_on_stream_empty(stream_pair.first);

            // 6. 暂停 rm_node 自身
            rm_node->wait_paused();

            // 7. 从上游节点的OutputStreamManager中移除连接到rm_node的MirrorStream
            for (auto upstream_node : paused_upstream_nodes) {
                 /*...*/ upstream_osm->remove_stream(/*...*/);
                 upstream_node->set_outputstream_updated(true); // 通知上游节点输出已更改
            }

            // (如果rm_node是源节点，则直接暂停并标记输出EOF)
            // else { /*...*/ output_stream_manager->probe_eof(); /*...*/ }

            // 8. 恢复上游节点
            for (auto node : paused_upstream_nodes) { /*...*/ node->set_status(NodeState::PENDING); }

            // 9. 等待 rm_node 的输出流被下游消费完毕
            for (auto stream_pair : output_stream_manager->output_streams_)
                output_stream_manager->wait_on_stream_empty(stream_pair.first);

            // 10. 暂停所有直接下游节点
            std::vector<int> downstream_node_ids;
            output_stream_manager->get_outlink_nodes_id(downstream_node_ids);
            std::vector<std::shared_ptr<Node>> paused_downstream_nodes;
            for (int ds_id : downstream_node_ids) {
                // ... (获取下游节点nd, 调用nd->wait_paused()) ...
                paused_downstream_nodes.push_back(nd);
                // 从下游节点的ISM中移除rm_node作为上游
                nd->get_input_stream_manager()->remove_upstream_nodes(id_of_rm_node);
            }

            // 11. 移除 rm_node 的所有输出流及其连接
            for (auto stream_pair : output_stream_manager->output_streams_) {
                 /*...*/ output_stream_manager->remove_stream(stream_pair.first, -1);
            }

            // 12. 从调度器中移除 rm_node
            scheduler_->add_or_remove_node(rm_node->get_id(), false);

            // 13. 关闭 rm_node 并从Graph中移除
            rm_node->close();
            nodes_.erase(rm_node->get_id());
            // ... (日志) ...

            // 14. 恢复下游节点
            for (auto node : paused_downstream_nodes) {
                // ... (恢复状态) ...
                // 如果下游节点因此变成了源节点，则加入调度器
                if (node->get_input_stream_manager()->input_streams_.size() == 0) {
                    node->set_source(true);
                    scheduler_->add_or_remove_node(node->get_id(), true);
                } else {
                    node->set_status(NodeState::PENDING);
                }
            }
        }
    }

    // --- 处理重置节点 (nodes_reset) ---
    if (nodes_reset.size()) {
        for (auto &node_config : nodes_reset) {
            // 1. 根据别名找到要重置的节点
            int id_of_reset_node = -1; /*...*/
            std::shared_ptr<Node> reset_node = nodes_[id_of_reset_node];
            // 2. 调用Node的方法，传递新的配置选项
            reset_node->need_opt_reset(node_config.get_option());
        }
    }
    BMFLOG(BMF_INFO) << "dynamic update done";
    return 0;
}

// 优雅关闭: 等待所有节点完成
int Graph::close() {
    {
        std::unique_lock<std::mutex> lk(con_var_mutex_);
        // 等待，直到关闭计数等于节点总数 或 调度器报告了异常
        if (closed_count_ != nodes_.size() && !scheduler_->eptr_)
            cond_close_.wait(lk);
    }

    // 关闭调度器 (除非调度器本身异常，避免二次关闭可能导致死锁)
    if (not exception_from_scheduler_)
        scheduler_->close();
    else
        std::cerr << "!!Coredump may occured..." << std::endl;

    g_ptr.clear(); // 从全局列表中移除自己
    // 如果调度器捕获了异常，重新抛出
    if (scheduler_->eptr_) {
        // ... (打印状态信息) ...
        std::rethrow_exception(scheduler_->eptr_);
    }
    return 0;
}

// --- GraphInputStream / GraphOutputStream 实现 ---
// (这部分相对简单，主要是调用关联的Manager的方法)

void GraphInputStream::add_packet(Packet &packet) {
    // 创建一个包含单个packet的队列
    std::shared_ptr<SafeQueue<Packet>> packets = std::make_shared<SafeQueue<Packet>>();
    packets->push(packet);
    // 调用关联的OutputStreamManager的propagate_packets方法将数据注入
    // 这里的stream_id=0是因为GraphInputStream通常只模拟一个输出流
    manager_->propagate_packets(0, packets);
}

void GraphOutputStream::poll_packet(Packet &packet, bool block) {
    // 调用关联的InputStreamManager的pop_next_packet方法拉取数据
    // 这里的stream_id=0是因为GraphOutputStream通常只模拟一个输入流
    packet = input_manager_->pop_next_packet(0, block);
    // 触发关联节点的调度检查 (因为消费了数据，上游可能可以继续发送)
    if (scheduler_) // 确保scheduler有效
         scheduler_->sched_required(node_id_, false);
}
// ... (其他方法实现) ...

END_BMF_ENGINE_NS // BMF引擎命名空间结束
```

