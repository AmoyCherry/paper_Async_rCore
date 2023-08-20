## 20 April 2023, Reviews from SEKE23

### ----------------------- REVIEW 1 ---------------------

SUBMISSION: 7
TITLE: A Design and Implementation of Rust Coroutine with priority in Operating System
AUTHORS: Weizhen Sun, Wenzhi Wang, Yong Xiang and Jingbang Wu

#### ----------- Overall evaluation -----------

SCORE: 0 (borderline paper)

#### ----- TEXT:

1. This paper presents an implementation of priority-scheduled coroutines for the Rust language. The English language used in this paper is a bit weak and hinders understanding at times. The subject is interesting but of little-to-no relevance to the scope of the conference.

   > 1. English writing is weak.
   > 2. The subject is irrelevant to this conference.

2. In one of their main novelty claims, the authors of this paper make a point of being the first coroutine implementation to implement priority scheduling. One of the main points of coroutines is to not use (or pay for) a scheduler at all and instead allow (or require) the application to schedule its coroutines itself. 

   The reader is left with the impression that the authors have thus developed a coroutine-based very lightweight microthreads system, but are conflating their terminology. If the cost of the scheduler were measured and shown to be bounded, or its complexity in time and space were discussed, this might resolve some questions.

   > If the paper is related to performance, then time and space complexity should be discussed.

3. The authors do not have to re-hash or defend their choice of a stackless architecture; they are popular these days. However, the reader is left wondering whether under the stackless architecture, some performance saved using coroutines is simply transferred over to additional time spent in memory management. A stackless implementation moves work from a stack to a heap; what are the costs of your implementation using this heap?

   > Theoretical analysis of the cost of coroutine should be detailed.

4. The primary evaluation of the system is a comparison of the authors'coroutine-based concurrency versus threads. It is a given that coroutines are much faster than threads under many circumstances, and how big the speedup is also depends on whether the threads implementation you are comparing against is optimal. Perhaps a statement about assumptions is in order, such as: you believe that threads have been implemented near-optimally in Rust, and that the only coroutine applications that you are concerned with fit certain constraints where your concurrency and prioritization primitives are relevant. The authors would do well to compare the cost of their coroutines against
   the cost of goroutines or other famous and widely-used coroutine implementations, or to compare their coroutine-vs.-thread speedup ratio with that found in other languages' coroutine-vs.-thread speedup ratios.

   > It is difficult to prove that coroutine as a kind of schedule unit is much better than others.
   >
   > In order to increase credibility, comparing to other coroutines like Goroutine is an effect way.

5. Given the comments above, the authors' presentation of their existing results needs to be clearer. Are the experiments the averages taken of multiple runs?

   > The experiments' setup should be detailed.

6. In Table I, why would threads initially be cheaper? Why are coroutines costs linear while threads are not? Since this is a pathological worst-case artificial benchmark, do the authors believe real workloads would have similar differences?

   > 1. III.C has discussed the concern.
   > 2. The experimental scenario is designed, which can't convince others that there are the same scene and results in the real world.
   >


7. Do the authors believe there exist any workloads under which threads perform better than coroutines, or should threads simply be removed from the language in favor of coroutines? Are the reported differences universally and
   intrinsically true of all thread implementations, or are they artifacts of Rust's current thread implementation?

   > 1. We should answer: Do coroutines perform better than threads, and should they be removed from the language? 
   > 2. We should discuss: The theory and if the results could be reproduced in other languages.

8. "The state-of-art languages" should read "Two state-of-the-art languages"

   > Grammar error.

9. There is a grammar problem (singular versus plural conflict) in the paragraph that begins **"Stackless coroutines are used"**. **It starts plural, and then**
   **switches to singular.**

   > Singular versus plural conflict in I.A.

10. The paragraph beginning "Performance, memory safety, asynchronous support" needs to be rephrased. The authors do not need to state opinions in a research paper. Instead they should state motivations and research questions and then present results and evaluate those findings.

    > 1. Do not state opinions in a research paper.
    > 2. State **motivations** and **research questions** and then present **results** and evaluate those **findings**.

11. Past tense sentence at end of section I.B. ("This paper introduced") should be present tense

    > Tense error.

12. Missing space after the period in "ID.Considering"

    > Typing error.

13. Reference to "it" in "Its design and implementation" is unclear.

    > Pronouns 'it' here should be replaced with nouns The Model's.

14. "bitmap, If" should read "bitmap. If"

    > Typing error.

15. The term "clock cycle" is used in such a way that it is unclear if you really mean a few nanoseconds (1/frequency) or whether you mean the interval between system clock ticks (used to be 55ms, maybe smaller now but still a much longer amount of time). "clock cycle" means a few nanoseconds to me, enough for one instruction or whatever, but what you describe happening in one clock cycle sounds like a lot more work than that.

    > The term "clock cycle" is ambiguous and should be specified.

16. Reference [4] is missing publisher (Uppsala University), and frankly, citing someone's written examination is a sketchy practice. What authority does the author being cited have, other than being a university student?

    > Don't cite papers from students at the university level.

17. References 6, 7, 10, 19 and 27 are missing publisher information.

    > References format error.

18. Reference [8] is missing publisher information, and frankly, citing a tutorial written by a couple of UCLA master's students is a sketchy practice. By what authority are they the experts on Go concurrency? It would be better to cite a primary references from someone who actually developed Go concurrency.

    > Don't cite papers from students at the university level.

19. Reference [12] is missing publisher and page number information

    > References format error.

20. Reference [14] should be upgraded to the actual publisher, possibly:
    Proceedings of the VLDB Endowment, Vol. 14, No. 3 ISSN 2150-8097.
    doi:10.14778/3430915.3430932

    > References format error.

21. Reference [28] has a bad hyphen in "Califor-nia"

    > References format error.

22. Capitalization of various references (e.g. conference names) is inconsistent.

    > References format error.



### ----------------------- REVIEW 2 ---------------------

SUBMISSION: 7
TITLE: A Design and Implementation of Rust Coroutine with priority in Operating System
AUTHORS: Weizhen Sun, Wenzhi Wang, Yong Xiang and Jingbang Wu

#### ----------- Overall evaluation -----------

SCORE: -1 (weak reject)

#### ----- TEXT:

1. The paper proposes a coroutine model and its implementation in the Rust language. An experimental section compares the proposed coroutine model with a thread-based solution and shows that coroutine behaves better especially with a high degree of concurrency. The paper si well-written even though the details about the implementation of the model are missing. Overall, the topic addressed in the paper is not new and the differences between the proposed coroutine model and the one already provided by Rust are not discussed. Moreover, the paper is quite out of the scope of the conference.

   > 1. The topic is not new.
   > 2. The difference between the model's coroutine and Rust's coroutine should be detailed and emphasized.
   > 3. The subject is irrelevant to this conference.



### ----------------------- REVIEW 3 ---------------------

SUBMISSION: 7
TITLE: A Design and Implementation of Rust Coroutine with priority in Operating System
AUTHORS: Weizhen Sun, Wenzhi Wang, Yong Xiang and Jingbang Wu

#### ----------- Overall evaluation -----------

SCORE: 0 (borderline paper)

#### ----- TEXT:

1. It’s not clear if the results reported in the paper can be replicated in a real-world commercial OS, as the experiments for performance comparison between coroutines and threads were simulated in rCore environment. How did you find a way to exclude the influence of some irrelevant background tasks in the simulated settings? The URL for [13] is not working. Algorithm 1 is actually not an algorithm, just an interface.

   > 1. The experimental scenario is designed, which can't convince others that there are the same scene and results in the real world.
   > 2. The reliability of the simulated experimental environment is low.
   > 3. Algorithm 1 is just a interface.

2. In terms of the choice of coroutines and threads, it helps if the authors can also discuss the types of applications or circumstances under which one is preferred over the other.

   > We should discuss: Why do we both need threads and coroutines?



## 摘要

1. 协程受欢迎的原因是，基于有限状态机的协程切换具有极小的切换开销。
    1. 第二句话有描述：“低开销的协作式切换是并发编程的有效工具”；
2. 你的协程模型可以实现抢占式切换，从而把低延时的线程切换和低开销的协程切换统一到一起了。是这样吗？
    1. 目前的情况是，在保留了线程上下文切换机制的基础上，允许协程的协作式切换。问题在于：
        1. 这种性质能否被描述为统一；
        2. 如果可以，那么目前的goroutine及其它用户态协程也是运行在一个线程中，它们都可以使用线程的上下文切换响应中断，这篇论文在这个问题上并没有推进；
## 引言

1. 第一段没有说清楚。由于线程切换不能复用栈，而协程切换可复用栈资源，从而协程在高并发情况下具有较少的栈资源开销。
    1. 第一段只是概括地描述协程切换不会引起栈切换，而线程的栈切换会成为高并发的瓶颈，引出协程这个值得研究的对象；
    2. 复用栈（动态绑定）的具体描述在2.1；
    3. 要给出复用栈这次描述，必然要先给出“协程包裹着一个异步函数，它运行在一个栈上”，否则只能概括为切换不引起栈切换；
2. 1.1节第一段的协程定义是包括用户线程和无栈协程这两种情况的。这是你想表达的吗？
    1. 1.1是引用的其它论文里对于协程的定义。
3. 1.1节第二段是想说协程与线程的区别吗？我没有搞清楚你想说什么。
    1. 是在描述二者区别。准备删除；
4. 1.2节第二段：什么是“惰性状态机”？
    1. 删除“惰性”二字；
5. 建议把1.2节的第一段和第三段合并成一段。
    1. 赞同，应该合并；
6. 关于rCore的介绍，需要同时说它的优点和缺点。优点是你选它的原因，缺点是你要改进的地方。
    1. 有思考过这个问题。但唯二能想到的优点是“实现简洁”，“文档健全”。
    2. 内存管理，进程管理，充分利用了rust语言的特征，便于异步的扩展、开发；
    3. 文章描写的角度并不是针对rcore进行改进，所以只提到它没有实现协程，而我实现了一个协程库，恰好可以用上去；没有选择去描述，rcore因为没有协程而导致的缺点。
7. 1.3节的第二个特点，需要在摘要中提到你的“调度算法”和“其他进程中协程状态的感知（拥有最高优先级协程的进程）”。
    1. 摘要中有描述：“并且通过引入内核优先级位图，使得内核可以通过优先级感知用户态的协程的存在，从而干预整个系统的协程调度。”
## 第二节

1. 2.1节的协程模型，需要有一个简要的描述“协程是一个与全局异步函数一一对应的处理机调度对象”。Rust语言提供与异步函数对象和对象的状态数据结构，协程库提供优先级字段，并封装成协程控制块（CCB）。内核维护的就绪协程最高优先级位图也应该在这里出现。
    1. 准备以此作为2.1节第一段，然后在介绍协程模型的各个部分；
2. 第一段：第一个介绍的应该是“协程模型”，而不是“模型的运行机制”。
    1. 我认为应该描述模型 + 运行机制；也就是在现在的基础上，在2.1之前通过架构图增加对于整个模型的各个部分描述；
3. 输入错误：“完成的协程机制”应该是“完整的协程机制”
    1. 已改正；
4. “异步函数因为主动让出引起的切换不需要保存上下文”描述不准确。应该是“主动让出时由于栈为空而不用保存栈的状态”。
    1. 栈不一定为空，只是相对于该异步函数来说，栈上已经没有了该异步函数的内容，这一点在“.....不需要保存上下文”之前已作出解释；
5.  建议把协程控制块的描述与协程执行过程的描述放在不同的段落中。
    1. 协程控制块的描述，加上补充的模型的描述，放到2.1开篇；
6. “动态绑定”没有说清楚。栈是处理机的资源；协程与栈绑定后形成线程并在处理机上执行。执行状态的协程主动让权时，解除绑定；执行状态的协程被抢占时继续占有栈，被视为线程切换。
    1. 动态绑定这一段只是对之前一段中所描述的现象的总结，前一段中有更仔细的描述；
    2. 不准备使用“形成线程”这个描述，容易带来不解：什么是“协程形成线程”，“协程数据结构与资源是否发生了变化”。
    3. “形成线程”其实是指协程可以使用栈执行自己的异步函数，以及可以使用线程的中断机制响应中断，在2.1第二段介绍协程的执行时，已经给出相应描述；“形成线程”是一个范围更大的描述；
7. 2.2节：协程接口参数中有优先级吗？
    1. 添加描述：“可以通过接口指定协程优先级，如果不指定，则为默认值”；
8. 2.3.1节：协程创建和协程启动的使用有什么限制吗？
    1. “任意线程的任意位置”都可以启动；
    2. “任意线程的任意位置”都可以创建；
9. 协程状态和变迁事件需要集中和完整地描述。
10. 2.4节的内容应该就是协程状态变迁描述中的一部分。这部分内容组织顺序还需要语音讨论后改进。
## 第三节

实验结果分析不仅要描述现象，还尝试解释原因。

随着并发的增大，线程的栈切换带来的性能减益影响越来越明显，而协程没有栈的切换、回收开销，所以并发增大时任务切换的影响小于线程。

对此我在第三节最后一段做了解释。


## 第四节

结论要呼应摘要和第一节的贡献描述，并强调关键的定量数据支持。

1. 第一段有再次概括贡献：
    1. 提出并实现了一个协程模型；
    2. 可以在内核态和用户态创建协程；
    3. 优先级，进程之内与进程之间；
2. 在第二段中补充实验的定量数据；





## 20221014-王文智的小论文第二稿修改

王文智、小论文、修改意见、

* 修改后的[第二版论文](https://github.com/AmoyCherry/papper_Async_rCore/blob/6e8909c80cc90f2ffe745044022999542946ea2a/draft.md)

### 向勇的修改意见

1. 2.1节第一段：管理协程的数据结构可以称为协程控制块（CCB, Coroutine Control Block）。
2. 建议在图1中对就绪协程的最高优先级位图有所表示。
3. 建议在2.2节的接口定义部分给出两个函数的完整接口定义。
4. 2.3.2.1节的位图描述需要一个插图，直观表示每个进程的位图和整个系统的位图的关系。
5. 2.3.2.2节中的内核位图应该与系统位图是两个东西。对吗？
6. 建议2.4节加一个图表示协程的状态和状态间的转换，类似于三状态进程模型的图。
7. 3.1节中的“把协程和线程统称为task”不恰当。“进程”也有人称为“任务（task）”。这里的描述有些乱，我没有太理解（特别是“第三步”）。我对这个实验步骤的理解是，在每个进程中分别创建若干协程或线程来执行一个操作序列（你说的task），然后统计执行时间。
8. 3.1节的“读操作”和“写操作”是从哪儿读多少数据？
9. 3.1节的实验中，可以说一下协程和线程方式下的并发量最大值信息。
10. 3.1节和3.2节的实验步骤描述还有些乱，请尝试简洁和清晰一些的描述。如在步骤描述中只描述步骤。
11. 把实验结果的插图加到文章中。



## 20221023-王文智的小论文第三稿修改

### 向勇的修改意见

1. 标题：A Design and Implementation of Rust Coroutine with priority in Kernel
2. 如果可能，1.1节的最后一段可以加一些参考文档。
3. 在论文中引用别人的信息都通常都需要标注出处（参考文献）。请检查全文的各种信息引用是否都标注了。如1.2节的第三段。目前的参考文献列表还差得比较多。
4. 1.3节开头可以一段简要描述论文工作的思路，以过渡到两点论文贡献。目前的版本中各部分间没有很好地形成一个整体。
5. 目前的中文稿还有比较多的文字描述不够流畅的问题，就到英文稿中处理吧。
6. 输入错误：就绪态度、还不不遵循优先级调度、
7. 2.3.2.1的第三段需要说明一下，内核有独立地址空间是rCore的特征。（别的操作系统可能不这样做）
8. 2.4节的插图中“dead”这个用词不规范，通常称为“exit”。
9. 所有插图需要有编号。如3.2节的多个插图，我不能准确了解文字的分析对应哪个图。
10. 直接在英文稿中进行上述问题的修改吧。
