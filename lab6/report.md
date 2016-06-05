# 练习一
##1. sched_class及Round Robin调度算法

在sched.h中，sched_class的定义如下：
```
    struct sched_class {
        const char *name;
        void (*init)(struct run_queue *rq);
        void (*enqueue)(struct run_queue *rq, struct proc_struct *proc);
        void (*dequeue)(struct run_queue *rq, struct proc_struct *proc);
        struct proc_struct *(*pick_next)(struct run_queue *rq);
        void (*proc_tick)(struct run_queue *rq, struct proc_struct *proc);
    };
``` 
其中，各个成员函数和变量的意义如下：
name是调度算法的名字
init是初始化函数
enqueue将一个进程加入就绪队列
dequeue将一个进程从就绪队列中删除
pick_next从就绪队列中选择下一个要运行的进程
proc_tick是相应timetick的处理函数。这个结构体的各个成员都是函数指针，具体在default_sched.c中实现。并在sched.c的sched_init函数里，通过如下语句进行绑定。

```
    sched_class = &default_sched_class;
```
Round Robin调度算法

下面以lab6默认的Round Robin为例，描述UCore的执行过程以及sched_class中各个函数指针的调用。

在lab6中，各个函数的调用点如下：

do_exit(proc.c中) 当用户线程执行结束后，退出，cpu控制权交出

do_wait(proc.c中) 进程进入挂起状态，等待子进程结束退出

init_main(proc.c中)第一个实际的内核线程initproc，第一个用户进程是通过initproc创建的子进程加载的。initproc会等待所有用户子进程结束，然后调用schedule函数

cpu_idle(proc.c中)idle内核线程不断查询就绪队列，找到就绪进程并调用schedule函数进行调度。

trap(trap.c中)，用户态进程被打断，如果前进程控制块的成员变量need_resched设置为1，则当前线程会放弃CPU控制权

在各个调度点，最终都将掉欧诺个sched.c中的schedule函数进行调度，schedule函数定义如下：
```
    void
    schedule(void) {
        bool intr_flag;
        struct proc_struct *next;
        local_intr_save(intr_flag);
        {
            current->need_resched = 0;
            if (current->state == PROC_RUNNABLE) {
                sched_class_enqueue(current);
            }
            if ((next = sched_class_pick_next()) != NULL) {
                sched_class_dequeue(next);
            }
            if (next == NULL) {
                next = idleproc;
            }
            next->runs ++;
            if (next != current) {
                proc_run(next);
            }
        }
        local_intr_restore(intr_flag);
    }
```
可以看出，该函数的主体部分都用关中断来保证是原子操作。首先，将当前进程的need_resched设置为0，若当前进程是运行状态，则调用sched_class_enqueue将其加入就绪队列。在UCore中，就绪和运行都用PROC_RUNNABLE来表示，可以区别的是，当前进程若为PROC_RUNNABLE态，则是运行态，而就绪队列中的进程为就绪态。然后调用sched_class_pick_next选出下一个进程。若成功选出，则将其从就绪队列中删除。如果选取失败，则说明此时已无实际的进程需要运行，那么就将cpu使用权交回给idleproc。如果选取的进程与当前不相同，则调用proc_run进行进程切换。

具体来看Round Robin算法，该算法就是轮流使用时间片。RR_enqueue函数，就是简单地将进程proc加到就绪队列的队尾，重置时间片，RR_dequeue也是简单第进行删除。RR_pick_next则是直接选取队列头元素，用le2proc(le, run_link);转换为proc然后返回。


##2.多级反馈队列调度算法概要设计    

多级反馈队列算法(MLFQ)大致有下列几个特点：
```
就绪队列被划分为多个子队列

每个子队列有不同的优先级

时间片大小随优先级别增加而增加

若进程在当前的时间片没有完成，则降到下一个优先级
```
根据上述特点，可以在sched.c中定义一个run_queue* 类型的数组，存储多个队列。并且可以实现不同的调度算法，在sched.c中定义sched_class *数组，用于绑定不同的调度函数。然后在run_queue结构体中，增加一个优先级变量uint32_t queue_priority,可以取消max_time_slice,再设置一个队列最小时间片min_time_slice,每次设置最大时间片时，设置为min_time_slice << queue_priority的大小。可以再在proc_struct结构体中，增加一个变量priority_of_queue，代表该进程目前所在的进程队列的优先级，根据这一变量的值，来判断该进程属于哪一个进程队列。对于当前正在运行的进程，它被调度时，会调用enqueue函数，可以将sche.c中的sched_class_enqueue等函数进行扩展，在一个函数里，根据进程所属队列的不同，来选择不同的调度算法。此时，“若进程在当前的时间片没有完成，则降到下一个优先级”这一特点可以在sched_class_enqueue函数中进行实现。在sched_class_enqueue对被调度的进程进行判断，如果进程的time_slice为0且不是僵尸状态(即尚未完成运行)，则将其从原本队列中删除并加入更高优先级的队列中。以上为大致的设计过程。


# 练习二

Stride Scheduling调度算法的设计实现过程，按照注释中的说明，分为如下几步骤：

##1.确定BIG_STRIDE

首先需要确定BIG_STRIDE,因为步进值PASS = BIG_STRIDE / Priority, 使用的都是整数，要保证优先级很大时，步进值不为0，则需要取BIG_STRIDE为一个较大的数。但是这里会涉及STRIDE溢出的问题。在实验指导书中提及，当BIG_STRIDE取得合适时，例如a，b两个STRIDE值有a==b，此时b加上其步进值溢出后得到b‘，那么仍有a-b’的有符号数小于0.因为BIG_STRIDE要尽可能大，则一定存在一个最大值，该最大值恰好满足上述条件。下面对该值进行推导。无符号整数的最大值为0xFFFFFFFF，现在设两个相同的STRIDE为a，b(a==b)。此时假设b加上其步进值得到b‘。因为a，b都是无符号整数，b溢出后得到b‘，要保证STRIDE算法正确运行，则此时需要满足a-b’<0.因为溢出时，a，b的大小和步进值可能有多种情况，a-b‘的有符号数小于0，即a-b’的无符号数大于0x7FFFFFFF。此时需要在b‘最接近a是也满足a-b’无符号数大于0x7FFFFFFF。当b步进值最大时，b‘最接近a，此时步进值为BIG_STRIDE.如上图所示，则此时满足如下关系式子：
```
    t1 + t2 = BIG_STRIDE
    a + t2 = 0XFFFFFFFF
    a - t1 > 0X7FFFFFFF
```    
解上述三个式子，可以得到：

    BIG_STRIDE <= 0x7FFFFFFF
    
所以能够得到BIG_STRIDE的取值应为0x7FFFFFFF

##2.srtide_init
 
初始化rq->run_list
设置就绪队列中进程数为0
设置lab6_run_pool为NULL。

##3.stride_enqueue

libs/skew_heap.h中定义的斜堆来使用，直接利用skew_heap_insert进行插入。
然后判断加入就绪队列的进程时间片是否用完，如果是，则重新设置为 rq->max_time_slice。
lab6_result此处还将时间片值是否大于 rq->max_time_slice页加入了判断，比较严谨，但是貌似不会出现这种情况。然后设置进程所处的就绪队列为rq，将rq的进程数加一。

##4.stride_dequeue

直接用skew_heap_remove删除页，然后将rq->proc_num减1即可。

##5.stride_pick_next

此处需要选取就绪队列中stride值最小的，因为使用了skew_heap，直接取堆顶元素，即rq->lab6_run_pool即可。
然后要改变stride，步进值为BIG_STRIDE / lab6_priority.如果priority为0，则步进值为BIG_STRIDE.

##6.stride_proc_tick

这里是对时钟的相应，每次调用则时间片减1，时间片用完，则设置proc->need_resched = 1，等待被调度。
这里我直接用run_timer_list();并且注意是每一次tick而不是之前的while loop

##7.priority.c

最后一个测试一直过不去，而且感觉本身代码没有什么问题。后来看priority.c，发现在注释里面，MAX_TIME这里需要大于1000ms。原因是priority.c其实上是fork了多个进程，然后让所有进程运行MAX_TIME的时间，进程实际上是在跑死循环，然后统计各个子进程做的循环的次数。各个子进程的优先级依次为1,2,3,4,5；按照STRIDE算法，操作系统为这些进程分配的时间与其优先级成正比，其比例，即“stride sched correct result”也应该是1,2,3,4,5.但原priority.c中定义的MAX_TIME为1000，这个时间过短，各个进程的执行时间难以体现处区别。我最后把MAX_TIME设置为18000时，基本能保证每次运行都出现1,2,3,4,5的比例。

    

