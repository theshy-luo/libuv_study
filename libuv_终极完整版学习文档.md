# libuv 终极完整版系统化学习文档

> 目标读者：具备 C 语言基础，希望深入理解 libuv 底层原理，并彻底掌握
> Node.js 事件循环实现机制。\
> 本文从架构 → 数据结构 → 事件循环 → I/O → Timer → 线程池 → async →
> 源码阅读 → 实战 → Node.js 关系，全链路展开。

# 1. libuv 总体架构

## 1.1 libuv 本质

libuv = 跨平台异步 I/O 抽象层 + 事件循环调度器。

核心职责：

-   提供统一的 Event Loop
-   封装 I/O 多路复用（epoll/kqueue/IOCP）
-   提供线程池补齐阻塞 API
-   提供跨平台抽象层

------------------------------------------------------------------------

## 1.2 分层结构

``` mermaid
flowchart TB
  JS[Node.js JS层] --> CPP[Node C++层]
  CPP --> UV[libuv API层]
  UV --> CORE[事件循环核心]
  CORE --> BACKEND[I/O 后端]
  CORE --> TP[线程池]
  BACKEND --> OS[操作系统]
```

------------------------------------------------------------------------

## 1.3 工作流程总览

``` mermaid
flowchart LR
  init --> register_handle
  register_handle --> submit_req
  submit_req --> uv_run
  uv_run --> poll
  poll --> callback
  callback --> uv_run
```

# 2. 核心数据结构

## 2.1 uv_loop_t

事件循环实例，所有 handle 和 request 都属于某个 loop。

关键概念字段：

-   active_handles
-   pending_queue
-   timer_heap
-   backend_fd
-   closing_handles

``` mermaid
graph TD
  Loop --> pending_queue
  Loop --> timer_heap
  Loop --> io_watchers
  Loop --> threadpool_queue
```

------------------------------------------------------------------------

## 2.2 handle vs request

  类型      生命周期   作用
  --------- ---------- -------------
  handle    长期存在   资源/事件源
  request   一次性     操作提交

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

libuv 不做 GC。

-   handle 由用户分配
-   request 通常栈/堆分配
-   close 后才能释放 handle

内存责任：**谁申请谁释放**

# 3. 事件循环原理（核心章节）

## 3.1 uv_run 结构

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
  G -->|no| H[exit]
```

------------------------------------------------------------------------

## 3.3 poll timeout 计算机制

``` mermaid
flowchart TD
  A[检查活跃句柄] --> B{是否存在?}
  B -->|否| EXIT[退出]
  B -->|是| C[检查timer]
  C --> D{是否有最近timer?}
  D -->|否| E[timeout=∞]
  D -->|是| F[timeout=最近差值]
  F --> POLL
  E --> POLL
```

------------------------------------------------------------------------

## 3.4 pending queue 机制

I/O、async、线程池完成 → 都进入 pending_queue。

统一调度，统一回调执行。

# 4. I/O 模型

## 4.1 Linux epoll

``` c
epoll_wait(epoll_fd, events, maxevents, timeout);
```

流程：

``` mermaid
flowchart LR
  注册fd --> epoll_ctl
  epoll_wait --> 返回就绪
  就绪 --> 加入pending
  pending --> 执行回调
```

------------------------------------------------------------------------

## 4.2 kqueue

基于 kevent，支持更丰富过滤器。

------------------------------------------------------------------------

## 4.3 Windows IOCP

``` mermaid
flowchart LR
  提交IO --> IOCP队列
  IO完成 --> Completion Port
  Completion --> 回调执行
```

IOCP 是完成通知模型，而非就绪通知模型。

# 5. Timer 实现机制

## 5.1 最小堆结构

libuv 使用最小堆管理 timer。

``` mermaid
graph TD
  A((5ms)) --> B((8ms))
  A --> C((12ms))
```

堆顶为最近到期 timer。

------------------------------------------------------------------------

## 5.2 工作流程

``` mermaid
flowchart TD
  插入timer --> heapify
  每轮循环 --> 检查堆顶
  到期 --> 执行回调
  未到期 --> 退出
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

编译：

    gcc demo.c -luv -o demo

# 6. 线程池设计

默认线程数：4（可通过 UV_THREADPOOL_SIZE 修改）

适用于：

-   文件 I/O
-   DNS
-   用户自定义 work

------------------------------------------------------------------------

## 6.1 工作流程

``` mermaid
flowchart LR
  主线程 --> work_queue
  work_queue --> worker
  worker --> 完成队列
  完成队列 --> async唤醒
  async --> 主线程回调
```

------------------------------------------------------------------------

## 6.2 示例

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

# 7. 异步机制（uv_async_t）

## 7.1 原理

跨线程通知主循环。

Linux：eventfd\
Windows：IOCP 通知

------------------------------------------------------------------------

## 7.2 流程图

``` mermaid
flowchart LR
  子线程 --> uv_async_send
  uv_async_send --> 写eventfd
  eventfd --> epoll唤醒
  唤醒 --> pending
  pending --> 回调
```

# 8. 源码阅读路线

推荐顺序：

1.  src/unix/core.c
2.  src/unix/linux.c
3.  src/timer.c
4.  src/threadpool.c
5.  src/async.c

阅读目标：

-   看 uv_run
-   看 uv\_\_io_poll
-   看 uv\_\_run_timers
-   看 uv_queue_work

# 9. 实战：完整 TCP Server

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

# 10. 与 Node.js 的关系

JS → C++ → libuv → OS

映射关系：

  JS API             libuv
  ------------------ ----------------
  setTimeout         uv_timer_t
  fs.readFile        uv_fs + 线程池
  net.createServer   uv_tcp_t
  process.nextTick   Node 内部机制

------------------------------------------------------------------------

## Node 事件循环与 libuv 对齐

``` mermaid
flowchart LR
  JS timers --> uv timers
  JS poll --> uv poll
  JS check --> uv check
```

# 终极总结

libuv =

-   单线程事件循环
-   I/O 多路复用封装
-   最小堆 timer
-   线程池补齐阻塞
-   async 跨线程唤醒
-   跨平台抽象

理解 libuv = 理解 Node.js 底层。
