# report

## [练习 0] 填写已有实验
本实验依赖实验1/2/3/4。请把你做的实验1/2/3/4的代码填入本实验中代码中有“LAB1”/“LAB2”/“LAB3”/“LAB4”的注释相应部
分。注意:为了能够正确执行lab5的测试应用程序,可能需对已完成的实验1/2/3/4的代码进行进一步改进。
```
	完成
```


## [练习 1] 完成练习0后,建议大家比较一下(可用kdiff3等文件比较软件)个人完成的lab5和练习0完成后的刚修改的lab6之间的区别,
分析了解lab6采用RR调度算法后的执行过程。执行make grade,大部分测试用例应该通过。但执行priority.c应该过不去
```
分析lab6采用RR调度算法后的执行过程（列出了几个主要函数及其作用）：
	1. RR_init()：初始化run_queue
	2. RR_enqueue()：进程进入就绪队列
	3. RR_dequeue()：使一个进程从就绪队列出队
	4. RR_pick_next() ：寻找下一个就绪的进程，由于RR内部采用的是FIFO算法，所以找到的是就绪队列中的第一个
	5. RR_proc_tick()：主要实现触发器，被timer函数调用，每次调用time_slice减一，直到减为0，设置其need_resched为1，表示
	在下一次因为中断而执行trap函数的时候，执行schedule函数，然后把当前进程放在就绪队列的队尾
	触发器，每次timer到时后，trap函数将会间接调用此函数来把当前执行进程的时间片

问答题：
	1.请理解并分析sched_calss中各个函数指针的用法，并结合Round Robin调度算法描ucore的调度执行过程
		各个函数指针的用法上面已经给出解答
		调度过程：
			关中断;
			if(当前进程就绪)
				sched_class_enqueue;//插入就绪队列
			sched_class_pick_next;//挑选一个可用的进程p
			if(p为空)
				p = idleproc;//将其设置为idleproc
			if(p不是当前进程)
				proc_run(p);//执行新进程p，进程切换
			开中断;
	2.请在实验报告中简要说明如何设计实现”多级反馈队列调度算法“，给出概要设计，鼓励给出详细设计
		2.1根据优先级的不同设置多个队列，队列的设计原则满足优先级越低时间片越长。
		2.2进程创建的时候默认进入最高优先级队列
		2.3如果进程的时间片用完，但是进程没有结束，将其插入到稍微低一层级的队列中，更改其时间片大小为当
			前队列时间片大小，从高优先级队列中选取一个可移植性的进程完成进程切换
		2.4为了提高查找不同优先级的队列的效率，可以将这些队列的首地址存储下来。
```


## [练习 2] 父进程复制自己的内存空间给子进程(需要编码)
```

实现过程和思路：
	1.设置BIG_STRIDE为0x7fffffff,即整型的最大值
	2.stride_init：初始化rq
		run_list 调用list_init
		lab6_run_pool = NULL
		proc_num = 0
	3.stride_enqueue：
		skew_heap_insert，将要入队的进程插入进程池，设置时间片大小
	首先将传入的proc插入到rp的进程池中（调用函数skew_heap_insert），然后根据情况设置proc的时间片大小，
	proc->rp = rq
	rq->proc_num ++
	4.stride_dequeue：
		调用skew_heap_remove将进程proc从进程池中删除并将其proc_num减一
	5.stride_pick_next：
		if（rq的进程池为空）//没有符合条件的进程
			return NULL;
		else
			取出进程池的首元素p
			调用e2proc将p转化为进程控制块
			然后修改p->stride
			返回p
	6.stride_proc_tick：
		进程的时间片减1
		如果减到了0
			need_resched = 1;//表示需要重新调度。
与参考答案的区别：
		比起答案，少了用list实现Stride Scheduling调度算法的过程，需要注意的是触发调用stride_proc_tick时，
	trap.c中需要调用sched_class_proc_tick函数，因为静态函数不能使用非静态变量，所以将该函数开头的static标
	志删掉，即tatic void改设为void
```
## 程序运行结果：
```
	可在lab6目录下result文件中看到。
			badsegment:              (2.0s)
		  -check result:                             OK
		  -check output:                             OK
		divzero:                 (1.8s)
		  -check result:                             OK
		  -check output:                             OK
		softint:                 (1.6s)
		  -check result:                             OK
		  -check output:                             OK
		faultread:               (1.5s)
		  -check result:                             OK
		  -check output:                             OK
		faultreadkernel:         (1.6s)
		  -check result:                             OK
		  -check output:                             OK
		hello:                   (1.9s)
		  -check result:                             OK
		  -check output:                             OK
		testbss:                 (1.7s)
		  -check result:                             OK
		  -check output:                             OK
		pgdir:                   (1.6s)
		  -check result:                             OK
		  -check output:                             OK
		yield:                   (1.6s)
		  -check result:                             OK
		  -check output:                             OK
		badarg:                  (1.7s)
		  -check result:                             OK
		  -check output:                             OK
		exit:                    (1.6s)
		  -check result:                             OK
		  -check output:                             OK
		spin:                    (1.7s)
		  -check result:                             OK
		  -check output:                             OK
		waitkill:                (2.2s)
		  -check result:                             OK
		  -check output:                             OK
		forktest:                (1.7s)
		  -check result:                             OK
		  -check output:                             OK
		forktree:                (1.6s)
		  -check result:                             OK
		  -check output:                             OK
		matrix:                  (14.3s)
		  -check result:                             OK
		  -check output:                             OK
		priority:                (11.8s)
		  -check result:                             OK
		  -check output:                             OK
		Total Score: 170/170
```

## 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点
```
	进程的调度策略体现（RR调度算法和stride scheduling调度算法的实现）、调度时机的选择
	进程的优先级实现
	进程调度的具体过程init、enqueue、dequeue、pick_next、proc_tick的实际实现
	斜堆的使用
```
## 列出你认为OS原理中重要的知识点，但在实验中没有对应上
```
	先来先服务算法、多级反馈队列调度算法、短进程优先调度算法、最高响应比优先算法、实时调度、多处理器调度和优先级反置
```




