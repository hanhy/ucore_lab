# report

## [练习 0] 填写已有实验
本实验依赖实验1/2/3/4/5/6。请把你做的实验1/2/3/4/5/6的代码填入本实验中代码中
有“LAB1”/“LAB2”/“LAB3”/“LAB4”/“LAB5”/“LAB6”的注释相应部分。并确保编译通过。注意:为了能够正确执行lab7的测试应
用程序,可能需对已完成的实验1/2/3/4/5/6的代码进行进一步改进。
```
	完成
```


## [练习 1] 理解内核级信号量的实现和基于内核级信号量的哲学家就餐问题(不需要编码)
	完成练习0后,建议大家比较一下(可用kdiff3等文件比较软件)个人完成的lab6和练习0完成后的刚修改的lab7之间的区别,
	分析了解lab7采用信号量的执行过程。执行make	grade,大部分测试用例应该通过。
### 请在实验报告中给出内核级信号量的设计描述,并说其大致执行流流程。
```
内核级信号量在ucorelab中表现为semaphore_t的结构体（内包含一个value整型变量和wait_queue进程队列）
相关函数包括sem_init、up、down和try_down（这里主要分析up和down函数）
	up函数：
		对应于V操作：关中断——>如果wait_queue为空，value++，开中断返回；如果不为空，查看进程等待的原因，如
	down函数：
		对应于P操作：关中断——>如果value>0，value--，开中断返回；如果value不大于0，将当前的进程加入到等待队列
		开中断切换进程（调用schedule函数）。如果被V操作唤醒，关中断然后从等待队列中删除与自身关联的wait再开中断
```
### 请在实验报告中给出给用户态进程/线程提供信号量机制的设计方案，并比较说明给内核级提供信号量机制的异同

```
实现了一个信号量的数据结构，与内核中的实现方法基本相同，为了实现两边信号量的同步，当构建了一个用户态的信
号量之后，内核态中同时也生成一个。将二者及其对应的进程关联起来。为相应的内核函数建立用户态接口（init、do
wn和up等）二者的操作就同步了。
同：都实现了同步互斥
异：用户态不能直接操作内核信号量
```

## 练习2: 完成内核级条件变量和基于内核级条件变量的哲学家就餐问题（需要编码）


### 实现思路

```
调用cond_signal判断count计数的大小
大于0：
	唤醒相应的进程
	令自身进入睡眠状态
	增加next_count计数
	当该进程被唤醒时
		next_count减一
否则
	直接返回。
cond_wait函数
	count计数加1
	根据next_count计数进行唤醒操作或者释放mutex
	自身睡眠
	当自身由于另一个进程的signal而被唤醒时
	继续执行
	被唤醒后
	count计数减1。
phi_take_forks_condvar函数：
	设置相应哲学家状态为饥饿状态
	调用phi_test_condvar函数检测该哲学家是否可以用餐
	如果函数处理完成后该哲学家的状态不是EATING
	说明该哲学家现在不能就餐
	令该哲学家进程进入等待状态
phi_put_forks_condvar函数：
	将该哲学家的状态更改为思考状态
	调用phi_test_condvar函数分别检测其左边和右边的哲学家能否就餐
```

### 与标准答案的不同

```
按照注释来写所以基本相同，但是有些时候考虑不够全面，比如错误处理和提示等等，已经按照答案加进去了。
```

### 请在实验报告中给出内核级条件变量的设计描述，并说其大致执行流流程

```
管程的数据结构在ucore_lab中定义为monitor_t，其组成结构如下所示：

typedef struct monitor{
    semaphore_t mutex;
    semaphore_t next;
    int next_count;
    condvar_t *cv;
} monitor_t;
下面介绍一下其中的变量的意义：
mutex保证每一次只允许一个进程进入管程
cv通过在合适的时机执行wait_cv函数，让条件C为真的进程离开管程进入睡眠状态，从而使其他的进程进入管程执行
next和next_count：
	当一个进程signal_cv时，会使几个进程进入睡眠状态，直到该进程离开管程其他的进程才能继续执行，其中next_count
	就是当一个进程发出signal_cv时进入睡眠状态的个数。
下面介绍条件变量结构condvar_t：
typedef struct condvar{
    semaphore_t sem;
    int count;
    monitor_t * owner;
} condvar_t;
其中sem是信号量，能实现让发出wait_cv且等待某个条件为真的进程处于睡眠状态，让发出signal_cv操作的进程通
过它唤醒睡眠的进程。
count表示等在这个条件变量上的睡眠进程的个数
owner表示此条件变量的宿主是哪个管程

cond_wait函数实现了wait_cv的操作
cond_signal函数实现了signal_cv的操作
```

### 请在实验报告中给出给用户态进程/线程提供条件变量机制的设计方案，并比较说明给内核级提供条件变量机制的异同。

```
建立一个用户态的数据结构，与内核态条件变量对应，建立该数据结构的一个实体的时候需要同时在内核态生成一个对应的
变量，建立二者的映射关系并存储、关联二者的进程。需要建立一些接口函数对应于初始化、wait操作和signal操作的系统
调用，便于用户态程序操作内核态的条件变量。

同：都实现了同步互斥机制
异：用户态不能直接操作条件变量，需要通过内核态提供的一系列接口对其做操作
```


## 程序运行结果：
```
	badsegment:              (4.7s)
  -check result:                             OK
  -check output:                             OK
divzero:                 (2.9s)
  -check result:                             OK
  -check output:                             OK
softint:                 (3.3s)
  -check result:                             OK
  -check output:                             OK
faultread:               (1.8s)
  -check result:                             OK
  -check output:                             OK
faultreadkernel:         (1.6s)
  -check result:                             OK
  -check output:                             OK
hello:                   (2.7s)
  -check result:                             OK
  -check output:                             OK
testbss:                 (1.8s)
  -check result:                             OK
  -check output:                             OK
pgdir:                   (2.7s)
  -check result:                             OK
  -check output:                             OK
yield:                   (2.8s)
  -check result:                             OK
  -check output:                             OK
badarg:                  (2.8s)
  -check result:                             OK
  -check output:                             OK
exit:                    (3.2s)
  -check result:                             OK
  -check output:                             OK
spin:                    (3.3s)
  -check result:                             OK
  -check output:                             OK
waitkill:                (3.8s)
  -check result:                             OK
  -check output:                             OK
forktest:                (2.9s)
  -check result:                             OK
  -check output:                             OK
forktree:                (3.2s)
  -check result:                             OK
  -check output:                             OK
priority:                (15.7s)
  -check result:                             OK
  -check output:                             OK
sleep:                   (11.7s)
  -check result:                             OK
  -check output:                             OK
sleepkill:               (3.0s)
  -check result:                             OK
  -check output:                             OK
matrix:                  (14.4s)
  -check result:                             OK
  -check output:                             OK
Total Score: 190/190

			
```

## 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点

```
信号量及其相关操作，状态变化
管程及条件变量
哲学家就餐问题
```

## 列出你认为OS原理中很重要，但在实验中没有对应上的知识点

```
原子操作
同步互斥方法的优化，死锁、饥饿。
```



