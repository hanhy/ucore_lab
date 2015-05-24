# report

## [练习 0] 填写已有实验
	本实验依赖实验1/2/3/4/5/6/7。请把你做的实验1/2/3/4/5/6/7的代码填入本实验中代码中
有“LAB1”/“LAB2”/“LAB3”/“LAB4”/“LAB5”/“LAB6”/“LAB7”的注释相应部分。并确保编译通过。注意:为了能够正确执行lab7的测试应
用程序,可能需对已完成的实验1/2/3/4/5/6/7的代码进行进一步改进。
```
	完成
```


## [练习 1] 完成读文件操作的实现(需要编码)
	首先了解打开文件的处理流程,然后参考本实验后续的文件读写操作的过程分析,编写在sfs_inode.c中sfs_io_nolock读文件
中数据的实现代码。
### 请在实验报告中给出设计实现”UNIX的PIPE机制“的概要设方案,鼓励给出详细设计方案
```
如果读取的开始位置没有对齐某个block的开始处，读取该block的数据：
	设置该block需要读取的内容的大小
	调用sfs_bmap_load_nolock函数得到磁盘编号
	调用sfs_buf_op函数进行读操作
	更新读取长度alen+=size
	如果nblks=0（读取结束），跳到out部分；
	更新存放数据位置buf+=size;读取开始块blkno++;剩余读取块数nblks--。
读取中间首尾对齐的块：
	设置读取的内容的大小size=SFS_BLKSIZE;
	调用sfs_bmap_load_nolock函数得到磁盘编号
	调用sfs_buf_op函数进行读操作
	更新已读取长度alen+=size、存放数据位置buf+=size、
	读取开始块blkno++、剩余读取块数nblks--。
如果结束地址并不是块结束的整数倍，表示不对齐，有剩余部分需要读取：
	得到剩余读取部分的长度size = endpos % SFS_BLKSIZE;
	调用sfs_bmap_load_nolock函数得到实际的磁盘编号
	调用sfs_buf_op函数进行读操作
	更新已经读取长度alen+=size
```

## 练习2: 完成基于文件系统的执行程序机制的实现(需要编码)
	改写proc.c中的load_icode函数和其他相关函数,实现基于文件系统的执行程序机制。执行:make	qemu。如果能看看到sh
用户程序的执行界面,则基本成功了。如果在sh用户界面上可以执行”ls”,”hello”等其他放置在sfs文件系统中的其他执行程
序,则可以认为本实验基本成功。
### 请在实验报告中给出设计实现基于”UNIX的硬链接和软链接机制“的概要设方案,鼓励给出详细设计方案

```
	为进程创建一个内存区域
	创建新的页表
	复制内容、数据、BSS：
		通过文件头部读取	
		通过程序头部读取
		调用mm_map函数去创建代码段与数据段的vma
		调用callpgdir_alloc_page函数去分配代码段、数据段的页，并复制文件的内容
		调用callpgdir_alloc_page函数去分配BSS，并将非配的页置零
	调用mm_map函数创建用户栈
	设置进程的内存空间、CR3、页表项
	设置用户栈参数
	设置用户栈中断帧trapframe
```

### 问答题
给出设计实现基于”UNIX的硬链接和软链接机制“的概要设方案，鼓励给出详细设计方案
```
硬链接：
	文件与inode直接相连。
	新建一个文件时，其inode与指向其目录的inode，目录inode引用计数加一；	
	删除一个文件时，目录inode引用次数减一，当且仅当inode的引用次数降为0时删除并回收inode和原文件
软链接：
	实质是指向另一个文件的间接指针，不受文件系统限制，是包含了另一个文件的路径名的文件。
	新建一个文件时，文件中存储目标路径，inode指定为链接类型。
	删除一个文件时，直接删除该链接文件及其对应的inode即可，不会影响到目标文件
```
## 程序运行结果
执行make qemu之后出现sh窗口，分别执行ls、hello输出如下结果：
```
ls
 
$ ls
 @ is  [directory] 2(hlinks) 23(blocks) 5888(bytes) : @'.'
   [d]   2(h)       23(b)     5888(s)   .
   [d]   2(h)       23(b)     5888(s)   ..
   [-]   1(h)       10(b)    40383(s)   softint
   [-]   1(h)       11(b)    44571(s)   priority
   [-]   1(h)       11(b)    44584(s)   matrix
   [-]   1(h)       10(b)    40391(s)   faultreadkernel
   [-]   1(h)       10(b)    40381(s)   hello
   [-]   1(h)       10(b)    40382(s)   badarg
   [-]   1(h)       10(b)    40404(s)   sleep
   [-]   1(h)       11(b)    44694(s)   sh
   [-]   1(h)       10(b)    40380(s)   spin
   [-]   1(h)       11(b)    44640(s)   ls
   [-]   1(h)       10(b)    40386(s)   badsegment
   [-]   1(h)       10(b)    40435(s)   forktree
   [-]   1(h)       10(b)    40410(s)   forktest
   [-]   1(h)       10(b)    40516(s)   waitkill
   [-]   1(h)       10(b)    40404(s)   divzero
   [-]   1(h)       10(b)    40381(s)   pgdir
   [-]   1(h)       10(b)    40385(s)   sleepkill
   [-]   1(h)       10(b)    40408(s)   testbss
   [-]   1(h)       10(b)    40381(s)   yield
   [-]   1(h)       10(b)    40406(s)   exit
   [-]   1(h)       10(b)    40385(s)   faultread
lsdir: step 4
$ hello
Hello world!!.
I am process 15.
hello pass.

```
使用make qemu 与 bash tools/grade.sh 命令后，输出如下：
```
badsegment:              (6.3s)
  -check result:                             OK
  -check output:                             OK
divzero:                 (3.4s)
  -check result:                             OK
  -check output:                             OK
softint:                 (2.8s)
  -check result:                             OK
  -check output:                             OK
faultread:               (1.8s)
  -check result:                             OK
  -check output:                             OK
faultreadkernel:         (1.6s)
  -check result:                             OK
  -check output:                             OK
hello:                   (3.3s)
  -check result:                             OK
  -check output:                             OK
testbss:                 (1.7s)
  -check result:                             OK
  -check output:                             OK
pgdir:                   (3.2s)
  -check result:                             OK
  -check output:                             OK
yield:                   (3.1s)
  -check result:                             OK
  -check output:                             OK
badarg:                  (3.1s)
  -check result:                             OK
  -check output:                             OK
exit:                    (2.8s)
  -check result:                             OK
  -check output:                             OK
spin:                    (3.1s)
  -check result:                             OK
  -check output:                             OK
waitkill:                (3.8s)
  -check result:                             OK
  -check output:                             OK
forktest:                (3.1s)
  -check result:                             OK
  -check output:                             OK
forktree:                (5.7s)
  -check result:                             OK
  -check output:                             OK
priority:                (15.6s)
  -check result:                             OK
  -check output:                             OK
sleep:                   (11.7s)
  -check result:                             OK
  -check output:                             OK
sleepkill:               (3.4s)
  -check result:                             OK
  -check output:                             OK
matrix:                  (16.4s)
  -check result:                             OK
  -check output:                             OK
Total Score: 190/190
```
结果正确，实验成功。

## 列出你认为本实验中重要的知识点，以及与对应的OS原理中的知识点。
```
文件、文件系统、文件描述符、文件别名、文件种类
虚拟文件系统VFS和简单文件系统SFS
文件的打开
```

## 列出你认为OS原理中重要的知识点，但在实验中没有对应上
```
冗余磁盘阵列和空闲空间管理
```



