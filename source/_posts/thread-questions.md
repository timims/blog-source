---
title: 高并发遥测数据查询结构
date: 2026-04-19 14:39:00
categories:
  - 并发
  - 查询
  - kafka
tags:
  - 并发
  - 线程
description: 针对高并发场景下的查询结构优化思考。
---

# 高并发遥测查询架构演进：从 Java 端 Future 聚合到 Kafka 端收齐

## 摘要

在多并发高频遥测查询场景下，我们最初采用了“Java 端并行查询 + `CompletableFuture.join()` 聚合 + 统一发送 Kafka”的方案。该方案实现直接、可读性好，但在并发升高后出现了明显的资源竞争和尾延迟放大。  本文复盘现状实现，分析瓶颈，并给出一套可落地的演进路径：**子查询完成后立即发送 Kafka，将收齐与聚合下沉到消息消费端**。核心结论是：这条路可行，但必须同时补齐幂等、超时补偿、状态追踪和可观测性。

---

## 1. 背景：为什么要改

我们的分析任务经常是“一个父任务 + 多个传感器子查询”。例如一个场景下 20 个传感器，就会拆成 20 个并行子任务。  当多个父任务同时提交时，问题开始显现：
1. 子任务数量快速膨胀，线程池和数据库连接池竞争激烈。  
2. 父任务需要等待所有子任务 `join()` 后才能进入下一步，尾延迟受最慢子任务影响。  3. Java 端聚合结果容器变大，内存压力上升。  这推动我们思考：

**能不能实现“查完即发”，不在 Java 端做大聚合？**

---
## 2. 现状方案复盘
当前链路可以概括为：
1. 接收父任务请求，生成 `task_id`。  
2. 按传感器拆分子查询，使用 `CompletableFuture` 并行执行。  
3. 每个子查询写入共享结果容器。  
4. 父任务 `join()` 等待所有子查询完成。  
5. 聚合后发送 Kafka，进入后续算法处理。  

这个方案在低并发下非常稳定，问题主要出现在高并发和大任务同时到达时。

---

## 3. `Future` 到底是什么
在这次架构讨论里，这个概念必须统一：
1. `Future` 不是线程，它是异步任务的回执对象。  
2. 任务提交后立即返回 `Future`，主流程可以先往下走。  
3. 调用 `get()` 或 `join()` 时才会等待任务完成。  
4. `Future` 最终要么给结果，要么抛异常。  
5. `CompletableFuture` 还支持链式编排，比如 `thenApply`、`thenCompose`。

---

## 4. 痛点定位
瓶颈不在“并行”，在“收口”并行查询本身没有问题，瓶颈在“父任务同步收口”：
1. 父任务必须等所有子查询完成，最慢那个决定整体耗时。  
2. 收口阶段集中占用内存与 CPU。  
3. 高并发下多个父任务同时收口，放大抖动。
所以改造方向不是减少并行，而是把收口位置从 Java 主流程迁移出去。

---

## 5. 改造目标：查完即发，Kafka 端收齐
目标链路：
1. Java 端子查询完成后立即发送 Kafka，不等待其他子任务。  
2. 消费端按 `task_id` 聚合并判定“是否收齐”。  
3. 收齐后再触发后续算法执行。  
这样做的收益是：
1. Java 端不持有大聚合容器，内存更稳。  
2. 父任务线程占用时间更短。  
3. 吞吐更线性，扩展点转移到 Kafka 消费组与状态存储。  

---

## 6. 协议与状态设计（核心）
要落地这个方案，消息协议必须先定好。建议至少包含：
1. `task_id`：父任务唯一标识。  
2. `sensor_id`：子任务维度。  
3. `batch_index`：分片序号。  
4. `total_batches`：总分片数。  
5. `expected_sensors`：任务期望传感器数。  
6. `payload`：本次查询数据。  
7. `event_time`：事件时间戳。  
收齐判定逻辑：
1. 对每个 `task_id` 维护 `received(sensor_id, batch_index)` 集合。  
2. 当所有 `sensor_id` 的分片都满足 `0..total_batches-1` 时，判定收齐。  
3. 超过超时窗口未收齐，走部分完成或失败补偿策略。

---

## 7. 关键实现建议
### 7.1 Java 生产端
1. 子查询完成后直接发送 Kafka。  
2. 失败可重试，但要带幂等键。  
3. 不再 `join()` 全量数据用于业务聚合，只做任务级状态回写。  

### 7.2 Kafka 消费端
1. 按 `task_id` 维护聚合缓存。  
2. 做去重：`task_id + sensor_id + batch_index`。  
3. 收齐后发“任务可执行”事件给算法调度器。 

### 7.3 状态存储
1. 轻量可先内存 + TTL。  
2. 生产建议 Redis，支持多实例共享状态。  
3. 最终态写回任务表：`RUNNING/SUCCESS/FAILED/TIMEOUT`。  

---

## 8. 风险与治理
对于该迁移方案，需要考虑如下问题：
1. 风险：消息重复。  
治理：幂等键 + 去重集合。  
2. 风险：消息乱序。  
治理：按分片索引判定，不依赖到达顺序。  
3. 风险：永远收不齐。  
治理：超时窗口 + 补偿任务 + 可观测告警。  
4. 风险：缓存膨胀。  
治理：TTL、最大任务数上限、清理线程。  

---

## 9. 如何验证
改造是否值得发布前至少对比以下指标：
1. 查询链路 P95/P99 延迟。  
2. 单机吞吐（task/s）。  
3. Java 服务内存峰值。  
4. 线程池排队时长。  
5. Kafka 端收齐率、超时率、重试率。  


---
## 10. 小结
“一个线程查完立刻发 Kafka，不在 Java 端聚合”这条路是可行的，尤其适合高并发和大任务场景。  但关键不在于“去掉 `join()`”，而在于**把聚合能力产品化**：协议、状态机、幂等、超时、补偿、监控，一个都不能少。  当这些配套齐全后，系统会从“同步收口型”演进为“消息驱动型”，稳定性和扩展性都会上一个台阶。

---

明白，那这一节就要“去业务化”，专注 Java 本身的基础与进阶能力，同时又能和你全文的技术深度匹配。建议写成**“工程级 Java 并发与运行时基础”**，而不是零散八股。

下面是一套**精简但有技术含量的附录内容**，可以直接放文末。

---

# 附录：Java 核心知识点（并发与运行时）

## 1. Java 并发模型基础

### 1.1 线程与进程

* 进程是资源分配单位，线程是执行单位
* Java 中线程由 JVM 映射到操作系统线程（1:1 模型）

### 1.2 线程状态

`NEW → RUNNABLE → BLOCKED / WAITING / TIMED_WAITING → TERMINATED`

重点理解：

* `BLOCKED`：等待锁
* `WAITING`：无期限等待（如 `Object.wait()`）
* `TIMED_WAITING`：带超时（如 `sleep()`）

---

## 2. Java 内存模型（JMM）

JMM 定义线程之间如何**可见性、有序性、原子性**。

### 2.1 三大特性

* **原子性**：基本操作不可分割（但复合操作不保证）
* **可见性**：一个线程修改，另一个线程能看到
* **有序性**：指令可能重排序

### 2.2 happens-before 规则（核心）

关键规则：

* 线程启动前 → `start()` 之后可见
* 线程结束 → `join()` 后可见
* `volatile` 写 → 后续读可见
* 解锁 → 加锁可见

本质：JMM 通过 happens-before 建立内存可见性约束

---

## 3. `volatile` 与 `synchronized`

### 3.1 `volatile`

保证：

* 可见性
* 禁止指令重排序

不保证：

* 原子性

适用场景：

* 状态标志位
* 单次写、多次读

---

### 3.2 `synchronized`

保证：

* 原子性
* 可见性
* 有序性

实现机制：

* 基于对象监视器（Monitor）
* JVM 层支持（偏向锁、轻量级锁、重量级锁）

---

## 4. `Lock` 体系（`java.util.concurrent.locks`）

相比 `synchronized`：

优势：

* 可中断（`lockInterruptibly`）
* 可超时（`tryLock`）
* 可公平锁

核心实现：

* `ReentrantLock`
* 基于 AQS（AbstractQueuedSynchronizer）

---

## 5. AQS（AbstractQueuedSynchronizer）

JUC 核心基础组件，支撑：

* `ReentrantLock`
* `CountDownLatch`
* `Semaphore`
* `FutureTask`

核心思想：

* 一个 `state` 状态变量
* 一个 FIFO 等待队列（CLH 队列）
* CAS + 自旋 + 阻塞

---

## 6. 线程池（ThreadPoolExecutor）

### 6.1 核心结构

```java
ThreadPoolExecutor(
  corePoolSize,
  maximumPoolSize,
  keepAliveTime,
  workQueue,
  handler
)
```

### 6.2 执行流程

1. 小于 core → 新建线程
2. 队列未满 → 入队
3. 队列满 → 扩容线程到 max
4. 再满 → 拒绝策略

### 6.3 拒绝策略

* AbortPolicy（默认）
* CallerRunsPolicy（反压）
* DiscardPolicy
* DiscardOldestPolicy

---

## 7. `Future` 与 `CompletableFuture`

### 7.1 `Future`

* 表示异步结果
* 通过 `get()` 阻塞获取

### 7.2 `CompletableFuture`

增强点：

* 链式编排
* 非阻塞回调
* 组合操作

常用方法：

* `thenApply`
* `thenCompose`
* `thenCombine`
* `allOf`

---

## 8. CAS（Compare-And-Swap）

无锁并发核心机制：

```text
if (V == A) then V = B
```

特点：

* 原子操作（CPU 指令级）
* 无锁（乐观锁）

问题：

* ABA 问题
* 自旋开销

---

## 9. 原子类（Atomic）

基于 CAS：

* `AtomicInteger`
* `AtomicLong`
* `AtomicReference`

高并发优化：

* `LongAdder`（分段累加，减少竞争）

---

## 10. 并发容器

### 10.1 `ConcurrentHashMap`

* 分段锁（JDK7）
* CAS + synchronized（JDK8）

特点：

* 高并发读写
* 线程安全

---

### 10.2 其他容器

* `CopyOnWriteArrayList`（读多写少）
* `BlockingQueue`（生产者-消费者模型）

---

## 11. 阻塞队列（BlockingQueue）

常见实现：

* `ArrayBlockingQueue`
* `LinkedBlockingQueue`
* `SynchronousQueue`

特点：

* 内置阻塞
* 支持生产消费模型

---

## 12. 线程通信机制

### 12.1 `wait / notify`

* 必须在 synchronized 中使用
* 基于对象监视器

### 12.2 JUC 替代方案

* `Condition`
* `CountDownLatch`
* `CyclicBarrier`

---

## 13. 并发工具类

### 13.1 `CountDownLatch`

* 一次性同步器
* 等待多个任务完成

### 13.2 `CyclicBarrier`

* 可复用屏障

### 13.3 `Semaphore`

* 控制并发数量

---

## 14. 内存与 GC 基础

### 14.1 内存区域

* 堆（Heap）
* 栈（Stack）
* 方法区（Metaspace）

### 14.2 GC 关注点

* 对象分配速率
* GC 停顿（STW）
* 大对象问题

---

## 15. 可见性问题的本质总结

并发问题本质来源于：

```text
CPU缓存 + 指令重排 + 多线程
```

解决手段：

* 锁（悲观）
* CAS（乐观）
* 内存屏障（volatile）

---

> Java 并发体系从 `synchronized` 到 AQS，再到 `CompletableFuture`，本质是在不断抽象“线程协作”的复杂性。但在高并发系统中，单机并发模型的优化终究有边界，系统往往需要借助消息队列与分布式状态来进一步解耦与扩展。

---
