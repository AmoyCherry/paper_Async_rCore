# A Coroutine Implementation of Operating System in Rust

> 贡献：
>
> 1. 我提出了一种Rust OS中的协程模型，利用Rust语言层面提供的异步函数封装出了协程；
> 2. 我添加了优先级机制，通过对优先级的控制，使得内核可以感知到协程的存在，使得系统总是能及时执行整个系统（内核态和用户态）中优先级最高的协程；
> 3. <u>通过VDSO向内核与用户态提供统一协程接口</u>（协程用法：低耦合，易用，对原代码的修改需求较小），并且利用内核协程来实现系统调用，对比与同步syscall，这可以<u>提高线程的并发程度</u>（线程+syscall会阻塞整个线程，协程+asyncall只会阻塞协程），并<u>减少内核执行系统调用的切换开销</u>；
>
> 2. 我在rcore中实现了这种模型；
> 3. 我设计并进行了实验得到结论；

内核调用库，映射到用户态

# Abstract 

1. ”协程模型“需要体现特点，与他人的差异，优先级、VDSO、共享；

## Keywords





# 1. INTRODUCTION

> 总结自己的工作

## 1.1 Coroutines

> 1. 简要介绍线程，然后引出协程，做对比，强调协程特性： stackless ，用figure辅助描述；（参考CoroBase）
> 2. 简单综述，介绍目前进展；



## 1.2 Rust and rCore OS

> 1. 简要介绍Rust作为<u>系统级软件</u>的开发语言的优势，以及对于异步的支持（异步机制介绍，async, await, waker）；（参考Rust官方文档）
>
> 2. 引出rcore，并指出目前的rcore没有充分利用rust得异步机制;



## 1.3 Contributions and Paper Organization



# 2.  COROUTINE MODEL

> coroutine model and implementation 
>
> 架构图

> 1. 描述架构图；
> 2. 通过优先级，协程可以被操作系统感知，从而实现灵活切换，在这一节可以描述各种调度情况和过程；
> 3. ~~用户态与内核态使用统一的调度代码；~~

## 2.1 Overview



## 2.2 Coroutine with Priority



## 2.3 Excutor and Asynchronization 

 

## 2.3 Scheduler

> Algorithm 1  Scheduler for Coroutine



# 3. IMPLEMENTATION and LIBRARY

> 1. ~~协程由async func封装为带优先级的coroutine，存储在堆上；（简短伪代码）~~
> 2. ~~Excutor管理coroutine，以及主动让出，唤醒过程；（figure）~~
> 3. ~~Scheduler 的工作流程；（figure）~~
> 4. 接口封装 add, run；
> 5. VDSO；



## 3.1 Interface of Library



## 3.2 VDSO



# 4. PERFORMANCE COMPARISON 

> 对rcore的改造；
>
> 实验一：不涉及内核态，完全在用户态的测试；
>
> 实验二：管道读写，有内核态的参与；



## 4.1 Experiment A



## 4.2 Experiment B



# V. CONCLUSION

1. 分析协程切换开销的减少，并且得到实验验证；



# REFERENCES