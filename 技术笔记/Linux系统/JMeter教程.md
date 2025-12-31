# JMter并发验证流程

## 场景需求：

1.1000个线程随机并行执行接口。

2.所有接口的task_id字段从5个预定义的task_ids池子中随机选择。

## 详细方案如下：

### 创建测试计划

#### 新建测试计划

1. 打开JMeter，默认会有一个测试计划（Test Plan）。
2. 右键点击Test Plan，选择添加->线程（用户）->线程组，添加一个线程组。

### 添加HTTP请求头

1.测试计划->添加->配置原件->HTTP信息头管理器，按需添加

![image-20250113150621922](G:\markdown-png\image-20250113150621922.png)

#### 配置线程组

1. 名称：例如task test

2. 线程数：1000

3. Ramp-Up：可以设置为60（即60秒内启动所有的线程，有助于避免瞬间大流量）

4. 循环次数：永久或者指定次数。

   ![image-20250113145014830](G:\markdown-png\image-20250113145014830.png)

### 定义任务ID变量

#### 添加用户自定义变量

1. 右键点击线程组，选择添加->配置元件->用户自定义的变量.

2. 名称：例如task_ids

3. 变量：

   ```
   task1,task2,task3,task4
   ```

![image-20250113145240681](G:\markdown-png\image-20250113145240681.png)

### 设置HTTP请求取样器

#### 添加接口选择器

1. 为了随机选择接口，可以使用随机控制器
2. 右键点击线程组，选择添加->逻辑处理器->随机控制器.

#### 增加任务ID选择器

1. 右键点击随机控制器，选择添加前置处理器JSR223 PreProcessor。

2. 名称：select task id

3. 脚本语言：Groovy

4. 脚本内容：从变量task_ids中随机选择一个id到selected_task_id

   ```groovy
   def taskIds = vars.get("task_ids").split(',')
   def selectedTaskId = taskIds[new Random().nextInt(taskIds.length)]
   vars.put("selected_task_id", selectedTaskId)
   ```

#### 在随机控制器下添加三个HTTP请求取样器

1. 接口1

   ```
   {
   	"task_id":"${selected_task_id}"
   	//other
   }
   ```

   

2. 接口2

   ```
   {
   	"task_id":"${selected_task_id}"
   	//other
   }
   ```

   

3. 接口3

   ```
   {
   	"task_id":"${selected_task_id}"
   	//other
   }
   ```

   ![image-20250113150705058](G:\markdown-png\image-20250113150705058.png)

   ### 添加监视器（可选）

   #### 聚合报告

   1. 线程组->添加->监视器->聚合报告

   #### 查看结果树

   1. 线程组->添加->监视器->聚合报告

   ####  汇总报告

   1.线程组->添加->监视器->汇总报告

​		![image-20250113150341898](G:\markdown-png\image-20250113150341898.png)

		### 测试执行

1. 保存测试计划，方便后续修改和复用。

2. 启动测试，点击工具栏上的绿色运行按钮。

3. 通过监视器来分析性能等关键指标。

4. 命令行

   ```shell
   jmeter -n -t your_test_plan.jmx -l results.jtl
   ```

   这条命令用于通过命令行运行Apache JMeter测试计划。各部分的含义如下：

   * jmeter：调用JMeter程序。
   * -n：指定JMeter以非图形用户界面（Non-GUI）模式运行。这种模式适用于压力测试和性能测试，因为它占用的资源较少。
   * -t your_test_plan.jmx：指定要执行的测试计划文件，这里是your_test_plan.jmx，.jmx文件是JMeter的测试计划文件，包含了所有的测试配置和脚本。
   * -l results.jtl：指定测试结果的输出文件，这里是results.jtl，测试完成后，结果会被记录到这个文件中，便于后续分析

   总结：这条命令的作用是在非图形界面模式下运行指定的JMeter测试计划，并将测试结果保存到指定的文件中。