---
title: brpc的bthread分析
date: 2020-09-20 11:42:51
tags: 
    -brpc
categories: 框架学习
---

## 前言
brpc是百度开源的一款rpc框架，因其高性能、扩展性强、在其基础上进行二次开发，可定制性强；国内许多公司有在使用该框架。brpc使用的是一个M:N的协程模型(bthread)，本文简单介绍一下这个模型。


<!-- more -->

## 模型架构
bthread的模型架构如下，M个bthread映射到N个pthread,pthread与内核线程的关系式1:1，bthread实际运行在pthread中，bthread运行一半之后可以放弃pthread资源，再次运行时可能处于不同的pthread.
{% asset_img bthread架构.bmp bthread架构 %} 

## bthread的简单使用

```cpp
std::vector<bthread_t> tids;
for (int i = 0; i < thread_num; ++i) {
    if (bthread_start_background(&tids[i], NULL, myfunc, &myarg) != 0) {
        LOG(ERROR) << "Fail to create bthread";
        return -1;
     }
}

for (int i = 0; i < thread_num; ++i) {
     bthread_join(tids[i], NULL);
}
```
- `bthread_start_background`相当于`pthread_create`,使用函数创建一个bthread.
- `bthread_join`相当于`pthread_join`等待协程的结束.

## bthread的原理
bthread主要接口函数和一些实现类构成。比较重要的类有两个，`TaskControl`和`TaskGroup`在bthread中一个协程也称作一个task.`TaskControl`负责管理所有的pthread.`TaskGroup`负责每个pthread的工作。
1. 接口函数：
`bthread_start_urgent`:创建一个协程，并立即执行该协程。
`bthread_start_background`：创建一个协程，并将其加入调度队列。
`bthread_yield`：放弃cpu的执行权，切换到另外的协程执行。

2. `TaskControl`的主要接口
`init`:bthread的初始化，其中会调用`pthread_create`创建工作线程。
`create_group`：创建一个`TaskGroup`对象。
`steal_task`：使bthread可以在不同的pthread中进行切换，提高并发度。
`choose_one_group`：随机选择一个管理的`TaskGroup`对象返回。

3. `TaskGroup`的主要接口
`sched_to`：切换到某个协程去执行。
`run_main_task`：工作线程执行的函数，主要工作为不停的选择不同的bthread并调度执行。

## 函数时序图
本文时序图省略了部分函数，保留整体逻辑。
### `bthread_start_background`时序图
{% asset_img bthread_start_background时序图.bmp bthread_start_background时序图 %}
1. `start_from_non_worker`：在不是worker的线程中调用该函数。
2. `TaskControl* c = get_or_new_task_control()`:获得`TaskControl`对象，如果不是第一次执行，就会执行`TaskControl::init`。
3. `TaskControl::init`:创建worker线程。
4. `TaskGroup* g = c->choose_one_group()`:随机选择一个`TaskGroup`。
5. `ready_to_run_remote`:将该bthread加入到worker线程的调度队列中。
### `run_main_task`时序图
{% asset_img run_main_task时序图.bmp run_main_task时序图 %}
1. `wait_task(&tid)`:调度一个可执行的bthread,可能会从其他worker偷取bthread。
2. ` TaskGroup::sched_to(&dummy, tid)`:调度指定bthread执行。
3. `jump_stack(cur_meta->stack, next_meta->stack);`:切换用户态的栈。




