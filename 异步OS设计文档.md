[共享调度器开发日志](https://shimo.im/docs/9RKPKYvJKpRv8hYc)

# 共享调度器方案

目标：

1. 支持优先级

2. 统一进程、线程和协程调度

3. 支持中断的及时响应
   
   # 异步OS的目标（针对这样的目标进行设计）

4. 调度器的最小调度单位、CPU的最小执行单位是协程；

5. 内核向用户态提供协程的使用接口；

6. 协程、线程、进程都支持优先级调度；

7. 协程可以支持强制切换和主动（协作式）切换，由内核选择；

# 一、数据结构

## 1.1、进程

进程提供地址空间与页表，进程的地址空间严格隔离。切换时有页表切换的开销

```rust
pub struct Process {
    pub id: ProcessId,
    // 进程地址空间/页表
    pub memory_set: MemorySet,
    ...
}
```

## 1.2、线程

每个线程有独立的堆栈,切换时需保存和恢复全部寄存器,视为在一个cpu上运行的抽象

```rust
pub struct Thread {
    pub id: ThreadId,
    // 标识当前线程所属的地址空间
    pub space_id: usize,
    // 线程上下文
    pub context: Context,
    // 线程堆栈
    pub stack: Stack,
    ...
}
```

## 1.3、协程

协程封装实际的执行内容（Future）。共用一个堆栈的状态机转移函数

```rust
pub struct Coroutine {
    pub id: TaskId,
    // 具体的任务内容
    pub future: Mutex<Pin<Box<dyn Future<Output=()> + 'static + Send + Sync>>>, 
    ...
}
```

# 二、内核态的任务调度

## 2.1 调度运行用户进程

```rust
pub struct TaskControlBlock {
    // immutable
    pub pid: PidHandle,
    pub kernel_stack: KernelStack,
    // mutable, context
    inner: Mutex<TaskControlBlockInner>,
}

pub struct TaskControlBlockInner {
    pub task_cx: TaskContext,
    pub task_cx_ptr: usize,

    pub memory_set: MemorySet,

    ...
}
```

1. 内核初始化完成之后，读取用户态程序编译为`elf文件`，并创建对应的进程控制块(Process Control block)，插入到进程队列中；

2. （多个或单个）处理器互斥访问进程队列，每次取出一个`PCB`，运行该进程；

## 2.2 调度运行内核态协程

内核使用的协程调度器（的代码）与用户态使用的是同一份，但在协程队列为空时，行为有所不同。在用户态中，队列为空时，直接退出调度器，返回到原先的main函数（主线程）中；内核态中，调度器不会退出，而是会调度其他进程或无限循环；

1. 环境初始化：查询调度器的接口函数的地址，如：`add_coroutine(...)`、`run_coroutine() `；

2. 创建协程，并通过接口函数插入到协程队列；

3. 执行`run`函数，进入到一个不断访问协程队列去获取协程的循环；

4. 内核协程执行完成之后，会调度运行用户进程，如果没有，则会进入循环，等待内核协程的添加；

### 2.2.1 创建内核协程的方式

1. 写死在代码中，内核初始化时创建；

2. 用户态通过系统调用主动创建一个协程并插入到内核的协程队列；（**目前不支持**）

3. 作为某一种机制的解决方案，如：`IO`多路复用时，内核中的每一个`IO`事件创建为一个协程；（**目前不支持**）

# 三、用户态的任务调度

此节描述的是每一个进程内部的任务调度。

## 3.1 理想的实现

1. 一个进程的main函数被创建为主线程，从这个主线程开始运行进程；

2. 主线程的执行过程中会创建协程到协程队列中；

3. 通过一个**函数调用**，进入到一个循环，不断从协程队列中取出协程执行，主线程同时用作**第一个协程执行器**；

4. 可以根据某种策略，在某些时机下可以创建额外的多个（协程执行器）线程去执行协程，而且，这些线程可以并行在多个处理器上；

5. 每一个协程的执行过程中，都可以创建协程添加到协程队列；

6. 所有协程执行完成，返回到原先的主线中继续执行；

## 3.2 目前的实现

**每个进程只有一个线程。**

1. 一个进程的main函数被创建为主线程，从这个主线程开始运行进程；

2. 主线程的执行过程中会创建协程到协程队列中；

3. 通过一个**函数调用**，进入到一个循环，不断从协程队列中取出协程执行；

4. <u>目前同一个地址空间的线程无法并行在多个处理器上，所以使用单线程；</u>

5. 每一个协程的执行过程中，都可以创建协程添加到协程队列；

6. 所有协程执行完成，返回到原先的主线中继续执行；

## 3.3 协程的执行过程

处理器执行一个又一个的函数，只要处理器在运转，就一定有一个栈支持函数的运行，也必然有一个上下文，随时准备撤下处理器并在之后重新运行。

协程也不例外。协程没有唯一的栈，也没有上下文结构，当一个协程运行时，会使用（绑定）其协程执行器线程的栈。多个协程在同一个线程中运行时，会共用同一个栈，也就没有了栈和上下文的创建、切换、回收的开销。

### 3.3.1 协程主动让出

协程返回Pending，表示不能继续往下执行，此时将其状态修改为`不可运行`，由waker将其唤醒（将其状态修改为可以运行）。然后从协程队列中取出下一个协程，复用当前的线程栈。

### 3.3.2 协程被强制中断

比如遭遇了时钟中断。

协程会借助与线程的栈和上下文，保存处理器的执行状态，相当于线程在执行一个函数时被中断。

# 四、时钟中断到来时更新协程信息

## 4.1 Linux O(1)调度算法

![图片](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/o1.png)

### 4.1.1 数据结构

1. bitmap：每个优先级使用一个bit，代表此优先级是否可以执行的任务（0无，1有），第一个优先级是bitmap的左起第一位，最低优先级是右边第一位。
   1. 需要两个bitmap，一个active bitmap，一个expired bitmap；
2. deque：每一个优先级后面都有一个队列，可以在O(1)的时间内取头/插尾。

### 4.1.2 算法主要操作

**1. pick_next_task()：**

1. 先找到active bitmap中左起第一个为1的bit，得到其位置索引 **x**；

2. 得到该优先级对应的队列 **active_priority_queue[x]**；

3. 取出队列头，得到一个当前系统中，可以执行的，优先级最高的任务;

**2. active_priority_queue[x].poll()**;

1. 如果active bitmap 中全为0，交换active合expired；expired bitmap的存在，主要是为了避免出现，低优先级任务长时间不被执行的情况；

### 4.1.3 算法参考

r-Core，只实现了1个优先级，即总的只有两个队列：[git link](https://github.com/rcore-os/rcore-thread/blob/master/src/scheduler/o1.rs)

## 4.2  协程调度设计

1. 位图的功能：提示哪一个优先级有可执行的协程，提示当前可以执行的最高优先级；

2. 内核维护全局的协程优先级信息，时钟中断到来时更新系统内的优先级信息，优先级信息保存为位图；

3. 协程退出（执行结束/主动让出），会使得当前线程的堆栈为空，可以调度下一个协程，此时去位图中搜索：

4. 如果当前可以运行的最高优先级协程不属于当前地址空间，则进行进程切换；

5. 考虑到切换的开销，可以设置一个机制：如果当前地址空间内存在可以执行的协程，且更高优先级的协程数量小于一个常量，则可以先调度执行当前地址空间内的协程；

## 4.3 协程调度实现

### 4.3.1 优先级位图

```rust
pub struct PrioMap(u64);

impl PrioMap {
  pub fn new() -> Self {
    Self {0:u64}
  }

  fn search_highest_prio() -> usize {
    /// bsf指令，找最左边第一个1
  }
}
```

### 4.3.2 进程的位图

```rust
pub struct Process {
  ...
  /// 当前进程内可以执行的协程的优先级分布
  prio_map: PrioMap,
  /// 内核位图
  global_map: usize;
}

impl Process {
  ...
  fn get_1st_prio(&self) -> usize {
    self.prio_map.search_highest_prio()
  }

  fn get_priomap_as_u64() -> PrioMap {
    self.prio_map.0
  }
}

// 进程初始化时，每个进程映射到一个相同的物理页，这个物理页存放着内核位图，只读,
// 每个进程保存这个页面虚拟地址，即保存内核位图为指针
```

### 4.3.3 内核的位图

时钟中断到来时更新：

1. 用户进程：内核位图 |= 所有用户进程位图；

2. 内核进程：内核协程增删时实时更新内核位图；

### 存在的问题

**用户态与内核位图不同步**：当最高优先级协程执行完时，该进程进行位图比较，发现本进程依然存在着当前系统内的最高优先级协程，但其实已经执行完成，也就是说，每个时钟中断到来时，会进入当前系统中拥有最高优先级协程的地址空间执行任务，但不能保证每次调度运行的协程都是系统中的最高优先级协程；

### 4.3.4 调度过程

从获取内核位图的最高优先级，与本进程内的最高优先级相比，如果本身不包含最高优先级，则让出；

**现在要扩展该算法对于异步的处理方式。**

异步其实就是需要多考虑一种情况：任务主动让出时如何处理？

- pick_next_task；

# 五、Waker

```rust
// 一般过程:
match Future::poll(future.as_mut(), &mut cx) {
    Poll::Ready(val) => break val,
    // 返回 Pending 时将协程状态修改为【不可执行】
    Poll::Pending => parker.park(),
};

// 比如我创建了一个协程，任务内容是一个长时间io，或是睡眠一定时间，此时会返回Pending,在睡眠结束后手动调用wake
let event_handle = thread::spawn(move || {
    thread::sleep(Duration::from_secs(duration));
    let reactor = reactor.upgrade().unwrap();
    reactor.lock().map(|mut r| r.wake(id)).unwrap();
});
```

## waker存在的问题-2022-03-31

### 问题背景

希望调度器支持这样的调度场景：多个IO任务和多个CPU计算任务，IO任务开始执行时，通过系统调用进入内核，等待IO，此时**返回Pending，修改此任务状态**为【不可被调度执行】，当IO处理完成后返回时，**Waker将任务状态修改为【可以执行】**，使得处理器所执行的每一个任务，都是可以继续向前执行的任务，这样做带来两个好处：

1. 处于等待状态的任务不会被调度，从而减少无意义的任务切换；

2. 任务开始IO时就马上返回Pending，为下一个任务让出处理器；
   
   ### 问题描述

任务返回Pending时可以拿到Reactor从而修改任务状态，但是，任务等待的事件完成时，如下代码所示，读函数返回，到了下一行代码，此时是在任务之中，无法拿到Reactor，也就无法修改任务状态。

```rust
// ----------------- 创建协程 -----------------
let task_inner = async {

    read_file("FILE_NAME").await; // 进入系统调用，期间不会被调度执行

    // 读结束，此时应该修改task状态为【可以执行】
    // !!! 但是这里拿不到Reactor

}

let task = COROUTINE::spawn( task_inner );

add_task_to_queue( task );

// ----------------- 协程执行器 -----------------
fn thread_main() {
    loop {
        let task = QUEUE.peek(); // 取出一个协程

        match task.task_inner.poll() { // 执行任务内容

            Pending => { 修改task状态为【不可被执行】 }

            Ready => { do_nothing }

        }
    }

}
```

由此引发的一个思考：在没有协程的时候，比如说Linux中，可以使用epoll进行io多路复用，由一个线程管理所有的io，随着各个io的完成，此线程被不断唤醒执行；在引入协程之后，只考虑一个进程（不考虑进程切换），那么被不断唤醒执行的就是各个协程，此时没有栈和context的切换，这是理论上性能可以提升的点。

# 六、具体的切换过程

## 5.1、内核协程之间的切换

## 5.2、同一用户进程内，用户协程之间的切换

## 5.3、不同用户进程内，用户协程之间的切换

## 5.4、内核协程与用户协程之间的切换

# 七、异步系统调用的实现方案

> 已有的工作：
> 
> 1. 通过共享内存使得内核可以访问用户程序的地址空间内的数据结构bitmap；

## 7.1 异步系统调用的过程

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/async_call.png)

## 7.2 回调

用户态中有一个数据结构---回调队列，用于保存已经完成异步系统调用任务的`TaskId`，通过共享内存由内核写入，也就是说，回调队列里每有一个可以执行回调函数唤醒的`TaskId`，必然是因为它提交的异步任务已经被内核完成。

1. 通过`TaskId`就可以在`Excutor`中找到对应的`Waker`；

2. 用户态的协程执行器中，每一轮循环的最后会判断回调队列是否为空；

## 7.3 数据结构

```rust
pub struct Excutor {
    ...
    pub waker_cache: BTreeMap<TaskId, Arc<Waker>>,
}

/// storage TaskId, we can use it to get its waker
pub struct CBQueue {
    tids: Vec<TaskId>,
}

/// 协程执行器
fn thread_main() {
    loop {
        // 取协程...
        // 执行...

        let cbq = CBQUEUE.lock();
        while !cbq.is_empty() {
            let tid = cbq.pop();
            let waker = EXCUTOR.get_waker(&tid);
            waker.wake();
        }
    }
}

// =============== KERNEL ===============
// 异步系统调用必须传递两个整数: space_id, TaskId as usize,
// space_id用于找到对应进程的回调队列, TaskId用于告知用户程序
// 已完成异步系统调用的协程

pub struct Event {
    space_id: usize,
    tid: TaskId,
    task: Task,
}

pub struct EventQueue {
    events: Vec<Event>,
}

// 内核中的协程执行器, 如果使用这样的设计, 协程在运行前需要判断当前是否
// 位于内核
fn thread_main() {
    loop {
        // 取一个事件
        let event = EVENTQUEUE.lock().pop();
        // 执行

    }
}

// 新的思路, 可以兼容用户态的thread_main
// 我需要的仅仅是在每个协程完成时访问用户进程的回调队列

pub fn async_sys_read(space_id: usize, 
                      tid: TaskId, 
                      path: String) 
{
    async fn work(...) {
        read(...);

        // 根据space_id找到对应地址空间的回调队列
        let cbq = ...;
        cbq.push(&tid);
    }

    add_task_with_prio(work, 0);
    // 系统调用返回
} 

// 问题是: 什么时候执行异步系统调用
// 1. 时钟中断到来时执行
// 2. 单独使用一个处理器进行处理
```

# 八、测试

## 8.1、基准测试

### 8.1.1、协程切换时间

单进程，单线程，跑`*`个空协程（协程不执行任何语句）。

| 协程数  | 总耗时 ms                      | 每次切换耗时 ms |
| ---- | --------------------------- | --------- |
| 501  | (67 + 58 +  75) / 3 = 67    | $0.134$   |
| 1001 | (142 + 131 + 148) / 3 = 141 | $0.114$   |
| 2001 | (214 + 229 + 257) / 3 = 233 | $0.116$   |
| 3001 | (415 + 407 +390 ) / 3 = 404 | $0.134$   |

## 8.2、 实验二：用户态，不涉及系统调用，协程、线程对比测试

- 如图所示，执行相同的任务内容，当并发量比较小时，协程的执行效率小于线程，当增大并发量时，协程的执行效率将大于线程，且差距逐渐增加

- 随着并发数的加大，线程的执行时间越来越长，但协程对高并发更耐受；

- 斜率代表单个任务的`平均执行时间 + 平均切换时间`，每个任务的执行内容相同，所以增加的是切换时间，可以看出，随着并发数的加大，线程的切换时间会大幅增加，而协程的增长接近一条直线，切换时间在所测试的尺度内没有明显增加

### 8.2.2 环境一  Mac OS上运行qemu

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/mac_tc.png)

### 

### 8.2.3 环境二 Linux上运行qemu

## 8.3、 实验三：利用管道读写模拟C/S通信，测量尾延迟

#### 8.3.1 环境一 MAC os 上运行qemu

##### 8.3.1.1 协程

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/c_percent.png)

##### 8.3.1.2 线程

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/t_percent.png)

##### 8.3.1.3 基于以上数据的统一比较

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/tc_percent.png)

1. 同一条曲线是固定任务数的情况下，不同的`任务完成数 / 总任务数 * 100%`的耗时的增长趋势；

2. 同一任务数的情况下，协程的耗时明显小于线程；

3. 同一任务数的情况下，线程的耗时的增长远大于协程的耗时的增长，这在图上体现为，线程的曲线远比协程陡峭；

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/tc_num.png)

1. 同一条曲线是固定百分比的情况下，不同协程数的耗时的增长趋势；

2. 随着任务数量的增加，各百分比的线程的的耗时远大于协程的耗时；

3. 任务数量的增加，对于线程的尾延迟有明显影响，且影响远大于协程，这在图上体现为，对于两组曲线左右两侧的差距，线程的左右差距明显更大；

## 8.4 实验四：管道连接任务为一个环，变量为环长度、数据量大小

### 结论

1. 从总体上来说，总任务数相同时，使用协程的总执行时间小于使用线程的总执行时间;

2. 随着任务数的增加，线程的总执行时间的增长更陡峭，相比之下协程更平缓，所以二者的总执行时间的差距随着任务数量的增加不断增大，当任务数量（并发数）到达4000时，线程的总执行时间约为协程的2倍；

3. `每个任务的平均执行时间 = 平均切换时间 + 平均调度时间 + 平均读一次管道时间 + 平均写一次管道时间`，如果认为任务内容的执行时间不会随着任务数量的增加而变化，那么，随着任务数量的增加，线程将快速增加切换和调度的时间，而协程则不受此影响，**所以在高并发的时间开销方面，协程表现出比线程更好的性能；**

4. 协程可以主动让出的特性的配合异步系统调用可以节省CPU的开销，因为协程不需要被大量轮询，只会在其等待的事件（异步系统调用）确定完成之后，才会被唤醒，这使得协程每次被唤醒执行都可以继续向前推进自己的工作。但是，异步系统调用需要维护一套回调机制，主要包括任务的提交以及回调的唤醒通知两部分，就目前的实现来说，大量的异步系统调用的使用会大量增加协程的执行时间；

### 8.4.1 1B

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/tc_1B.png)

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/avg_tc_1B.png)

### 8.4.1.1 线程

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/t_1B.png)

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/avg_t_1B.png)

### 8.4.1.2 协程

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/c_1B.png)

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/avg_c_1B.png)

### 8.4.2 256 B

![tc_256B.png](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/tc_256B.png)

![avg_tc_256B.png](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/avg_tc_256B.png)

#### 8.4.2.1 线程

![t_256B.png](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/256B.png)

![avg_t_256B.png](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/avg_t_256B.png)

#### 8.4.2.2 协程

![c_256B.png](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/c_256B.png)

![avg_c_256B.png](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/avg_c_256B.png)

### 8.4.3 4096 B = 4 KB

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/tc_4096B.png)

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/avg_tc_4096B.png)

#### 8.4.3.1 线程

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/t_4096B.png)

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/avg_t_4096B.png)

#### 8.4.3.2 协程

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/c_4096B.png)

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/avg_c_4096B.png)

### 8.4.4 通过修改协程优先级，减少协程的主动让出与被唤醒的次数

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/c_1B_without_prio.png)

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/c_256B_without_prio.png)

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/c_4096B_without_prio.png)

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/avg_c_1B_without_prio.png)

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/avg_c_256B_without_prio.png)

![](https://github.com/AmoyCherry/UnifieldScheduler/blob/main/src/avg_c_4096B_without_prio.png)

## 8.5 实验五: 内核中进行实验四

# #存在的问题

1. 协程执行前，用户态程序查询调度器接口的时间太慢；

# #踩坑记录

## 1、rcore的可用物理页数量

修改 `os/src/config.rs`中的`MEMORY_END`，在最后添加一个`0`即可，不能改得太大；

## 2、Heap allocation error 栈太小

- 用户态
  
  - `user/src/lib.rs   USER_HEAP_SIZE`

- 内核态
  
  - `os/src/config.rs   KERNEL_HEAP_SIZE`

## 1、协程创建过程中的栈分配情况

1. 内核初始化完成，加载用户程序为elf文件，并创建对应的进程控制块；

2. 启动进程调度，执行用户程序，每个进程从主函数main开始至执行，但是，执行main函数没有相应的线程控制块，但是有一个栈去支持main的执行，所以用户进程的主函数是一个**没有线程控制块**描述的、但是**具有栈**的线程。

3. 创建、添加协程到协程队列；

4. main函数中调用`CPU.run()`，初始化协程的执行环境并启动协程调度器执行协程：
   
   1. 创建线程池，包含若干个空线程与一个分配好栈的idle线程；
   
   2. `switch_to`切换到idle线程(idle使用main的栈---**函数调用** or 重新创建一个栈---**线程切换**)；
   
   3. 所有协程执行完毕，返回main函数；

# 参考资料

## 1、一种异步回调机制的实现 600行程序

1. 主线程注册并发送`读文件事件`，不去调用文件系统的接口进行`sys_read`，仅仅是注册事件；

2. 4个handle线程类似于内核，不断接受发送过来的`读文件事件`并且读取文件，读取完成之后，发送完成事件给主线程，告知事件已经完成；

3. 主线程在发送完所有的事件之后会进入到一个循环，不断轮询完成事件，并调用对应的回调函数进行收尾工作；
