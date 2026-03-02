# libuv 出版级完整版技术手册

## ------ 从架构设计到源码级实现的全链路深度解析

> 面向读者：\
> ✔ 具备 C 语言基础\
> ✔ 希望深入理解 libuv 底层原理\
> ✔ 希望彻底掌握 Node.js 事件循环底层实现

# 第一部分：总体架构

# 1. libuv 总体架构

## 1.1 libuv 的设计目标

libuv 是一个跨平台的异步 I/O 库，其设计目标包括：

-   构建统一的事件循环模型
-   屏蔽不同操作系统 I/O 模型差异
-   提供线程池补齐阻塞 API
-   为 Node.js 提供底层驱动

libuv 的本质：

> 单线程事件循环 + I/O 多路复用封装 + 线程池补齐 + 跨平台抽象层

------------------------------------------------------------------------

## 1.2 分层结构图

``` mermaid
flowchart TB
  JS[Node.js JavaScript层] --> CPP[Node.js C++ Binding层]
  CPP --> UV[libuv API层]
  UV --> CORE[事件循环核心]
  CORE --> BACKEND[I/O 后端实现]
  CORE --> THREADPOOL[线程池模块]
  BACKEND --> OS[操作系统内核]
```

------------------------------------------------------------------------

## 1.3 工作流程总览

``` mermaid
flowchart LR
  初始化 --> 注册Handle
  注册Handle --> 提交Request
  提交Request --> uv_run
  uv_run --> Poll阶段
  Poll阶段 --> 回调执行
  回调执行 --> uv_run
```

# 第二部分：核心数据结构

# 2. 核心数据结构

## 2.1 uv_loop_t

### 概念讲解

uv_loop_t 是整个 libuv 系统的核心调度器。

它负责：

-   管理所有 handle
-   管理所有 request
-   调度 I/O 事件
-   管理 timer 最小堆
-   管理 pending 队列

------------------------------------------------------------------------

### 结构示意图

``` mermaid
graph TD
  Loop --> pending_queue
  Loop --> timer_heap
  Loop --> io_watchers
  Loop --> closing_queue
  Loop --> threadpool_queue
```

------------------------------------------------------------------------

### 工作流程

1.  初始化 loop
2.  注册 handle 到 loop
3.  提交 request
4.  调用 uv_run()
5.  进入循环调度

------------------------------------------------------------------------

## 2.2 handle 与 request 的本质区别

  类型      生命周期   示例             作用
  --------- ---------- ---------------- ----------
  handle    长期存在   socket, timer    表示资源
  request   一次性     write, connect   表示操作

------------------------------------------------------------------------

### 生命周期图

``` mermaid
stateDiagram-v2
  [*] --> init
  init --> active
  active --> closing
  closing --> closed
  closed --> [*]
```

------------------------------------------------------------------------

## 2.3 内存模型

-   libuv 不做 GC
-   handle 由用户分配
-   close 回调后才能释放 handle
-   request 一般由调用者释放

原则：

> 谁分配，谁释放。

# 第三部分：事件循环原理（核心章节）

# 3. 事件循环深入解析

## 3.1 uv_run 的本质

核心结构：

``` c
while (loop_alive) {
    run_timers();
    run_pending();
    run_idle();
    io_poll();
    run_check();
    run_closing();
}
```

------------------------------------------------------------------------

## 3.2 阶段流程图

``` mermaid
flowchart TB
  A[timers] --> B[pending]
  B --> C[idle/prepare]
  C --> D[poll]
  D --> E[check]
  E --> F[close]
  F --> G{loop alive?}
  G -->|yes| A
  G -->|no| exit
```

------------------------------------------------------------------------

## 3.3 Poll 阶段详解

### timeout 计算逻辑

``` mermaid
flowchart TD
  A[检查活跃句柄] --> B{是否存在?}
  B -->|否| EXIT[退出]
  B -->|是| C[检查最近timer]
  C --> D{存在timer?}
  D -->|否| E[timeout=∞]
  D -->|是| F[timeout=最近差值]
  F --> POLL
  E --> POLL
```

------------------------------------------------------------------------

## 3.4 pending_queue 机制

所有 I/O、async、线程池完成都会进入 pending_queue。

统一执行顺序：

-   FIFO 执行
-   避免递归调用

# 第四部分：I/O 多路复用模型

# 4. I/O 模型

## 4.1 epoll 模型

``` c
epoll_wait(epoll_fd, events, maxevents, timeout);
```

流程：

``` mermaid
flowchart LR
  注册fd --> epoll_ctl
  epoll_wait --> 就绪事件
  就绪事件 --> 加入pending
  pending --> 执行回调
```

------------------------------------------------------------------------

## 4.2 kqueue 模型

-   基于 kevent
-   支持更多过滤器类型

------------------------------------------------------------------------

## 4.3 IOCP 模型

``` mermaid
flowchart LR
  提交IO --> IOCP
  IO完成 --> CompletionPort
  CompletionPort --> 回调执行
```

IOCP 属于完成通知模型。

# 第五部分：Timer 实现机制

## 5.1 最小堆结构

``` mermaid
graph TD
  A((5ms)) --> B((8ms))
  A --> C((12ms))
```

堆顶为最近到期 timer。

------------------------------------------------------------------------

## 5.2 执行流程

``` mermaid
flowchart TD
  插入Timer --> heapify
  每轮循环 --> 检查堆顶
  到期 --> 执行回调
  未到期 --> 结束
```

------------------------------------------------------------------------

## 5.3 示例代码

``` c
#include <uv.h>
#include <stdio.h>

void on_timer(uv_timer_t* handle) {
    printf("Timer fired\n");
}

int main() {
    uv_loop_t* loop = uv_default_loop();
    uv_timer_t timer;

    uv_timer_init(loop, &timer);
    uv_timer_start(&timer, on_timer, 1000, 1000);

    uv_run(loop, UV_RUN_DEFAULT);
    return 0;
}
```

# 第六部分：线程池设计

## 6.1 线程池原理

-   默认 4 个线程
-   使用全局工作队列
-   worker 线程阻塞等待任务

------------------------------------------------------------------------

## 6.2 工作流程图

``` mermaid
flowchart LR
  主线程 --> work_queue
  work_queue --> worker_thread
  worker_thread --> 完成队列
  完成队列 --> async唤醒
  async唤醒 --> 主线程回调
```

------------------------------------------------------------------------

## 6.3 示例代码

``` c
#include <uv.h>
#include <stdio.h>

uv_work_t req;

void work(uv_work_t* req) {
    printf("working in thread\n");
}

void after(uv_work_t* req, int status) {
    printf("done\n");
}

int main() {
    uv_loop_t* loop = uv_default_loop();
    uv_queue_work(loop, &req, work, after);
    uv_run(loop, UV_RUN_DEFAULT);
    return 0;
}
```

# 第七部分：异步机制（uv_async_t）

## 7.1 原理

跨线程通知主 loop。

Linux：eventfd\
Windows：IOCP 通知

------------------------------------------------------------------------

## 7.2 流程图

``` mermaid
flowchart LR
  子线程 --> uv_async_send
  uv_async_send --> 写eventfd
  eventfd --> epoll唤醒
  epoll唤醒 --> pending
  pending --> 回调
```

# 第八部分：源码阅读路线

推荐阅读顺序：

1.  src/unix/core.c
2.  src/unix/linux.c
3.  src/timer.c
4.  src/threadpool.c
5.  src/async.c

重点函数：

-   uv_run
-   uv\_\_io_poll
-   uv\_\_run_timers
-   uv_queue_work

# 第九部分：实战示例

## 9.1 TCP Server

``` c
#include <uv.h>
#include <stdlib.h>
#include <stdio.h>

uv_tcp_t server;

void on_new_connection(uv_stream_t* server, int status) {
    printf("New connection\n");
}

int main() {
    uv_loop_t* loop = uv_default_loop();
    uv_tcp_init(loop, &server);

    struct sockaddr_in addr;
    uv_ip4_addr("0.0.0.0", 7000, &addr);
    uv_tcp_bind(&server, (const struct sockaddr*)&addr, 0);

    uv_listen((uv_stream_t*)&server, 128, on_new_connection);

    uv_run(loop, UV_RUN_DEFAULT);
    return 0;
}
```

# 第十部分：与 Node.js 的关系

## 10.1 调用链

``` mermaid
flowchart TB
  JS API --> C++ Binding
  C++ Binding --> libuv
  libuv --> OS
```

------------------------------------------------------------------------

## 10.2 对应关系

  JS API             libuv
  ------------------ ----------------
  setTimeout         uv_timer_t
  fs.readFile        uv_fs + 线程池
  net.createServer   uv_tcp_t

------------------------------------------------------------------------

# 终极总结

libuv 是一个精密的事件调度系统。

理解 libuv =

-   理解单线程事件循环
-   理解 I/O 多路复用
-   理解最小堆 timer
-   理解线程池唤醒机制
-   理解 Node.js 底层执行模型

------------------------------------------------------------------------

（出版级完整版结束）
