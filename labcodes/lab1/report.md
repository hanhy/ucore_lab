# report 

## [练习 1] 理解通过make生成执行文件的过程

[练习1.1] 操作系统镜像文件 ucore.img 是如何一步一步生成的?(需要比较详细地解释 Makefile 中
每一条相关命令和命令参数的含义,以及说明命令导致的结果)

```
>ucore.img
	根据makefile文件的注释，我们可以看到“create ucore.img”，这一段下面的代码就是负责生成ucore
	.img的。生成ucore.img的代码就是
		# create ucore.img
		UCOREIMG	:= $(call totarget,ucore.img)

		$(UCOREIMG): $(kernel) $(bootblock)
			$(V)dd if=/dev/zero of=$@ count=10000
			$(V)dd if=$(bootblock) of=$@ conv=notrunc
			$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

		$(call create_target,ucore.img)
	分析这一段代码，注意到其中的一行：$(UCOREIMG):$(kernel) $(bootblock)，由此我们得出生成uco
	re.img的话，kernel和bootblock是必需的，下面分别介绍这二者的生成过程。
kernel
	向上查找代码，可以找到“create kernel target”的注释，这一段就是用来生成kernel的代码，如下：
		# create kernel target
		kernel = $(call totarget,kernel)

		$(kernel): tools/kernel.ld

		$(kernel): $(KOBJS)
			@echo + ld $@
			$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
			@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
			@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > 				$(call symfile,kernel)

		$(call create_target,kernel)
	可以发现，生成kernel首先需要kernel.ld，这个文件已经存在在tools文件夹里面；然后kernel还需要一
	些.o文件。执行make,发现kern文件夹下面生成的.o文件包括：kdebug.o、kmonitor.o、panic.o、cloc
	k.o、console.o、intr.o、picirq.o、init.o、readline.o、stdio.o、pmm.o、trap.o、trapentr
	y.o、vectors.o，然后加上libs文件夹下面的printfmt.o和string.o，这些就是生成kernel需要的.o文
	件。编译相关.c文件生成.o文件在makefile中的代码是：
		$(call add_files_cc,$(call listf_cc,$(LIBDIR)),libs,)
	这些.o文件在生成的时候基本过程都是一样的，仅有所需的.c文件文件名和目录不同，比如说kern/driver/
	clock.o，实际上进行编译的命令是
		gcc -Ikern/driver/ -fno-builtin -Wall -ggdb -m32
			-gstabs -nostdinc  -fno-stack-protector
			-c kern/driver/clock.c
			-o obj/kern/driver/clovk.o
	生成kernel的过程必须用到的代码是
		$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	实际运行的命令是：
		d -m    elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel \
		obj/kern/init/init.o obj/kern/libs/readline.o \
		obj/kern/libs/stdio.o obj/kern/debug/kdebug.o \
		obj/kern/debug/kmonitor.o obj/kern/debug/panic.o \
		obj/kern/driver/clock.o obj/kern/driver/console.o \
		obj/kern/driver/intr.o obj/kern/driver/picirq.o \
		obj/kern/trap/trap.o obj/kern/trap/trapentry.o \
		obj/kern/trap/vectors.o obj/kern/mm/pmm.o \
		obj/libs/printfmt.o obj/libs/string.o
	-o后面跟的是所需要的.o文件，该命令中-T用来指定连接器使用的脚本。
bootblock
	不难在makefile文件中找到生成bootblock的程序段：
		# create bootblock
		bootfiles = $(call listf_cc,boot)
		$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

		bootblock = $(call totarget,bootblock)

		$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
		@echo + ld $@
		$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
		@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
		@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
		@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

		$(call create_target,bootblock)
	根据make的运行结果：
		+ cc boot/bootasm.S
		+ cc boot/bootmain.c
		+ cc tools/sign.c
		+ ld bin/bootblock
	发现在生成bootblock的过程中需要boot/bootmain.c、boot/bootasm.S和tools/sign.c，运行make之
	后生成bootmain.o、bootasm.o和sign，下面分别介绍三个文件的生成过程：
	
	bootmain.o
	该文件生成的过程如同kernel中的其他由.c文件生成.o文件的过程类似，不再赘述。
	bootasm.o
	该文件由bootasm.S文件生成，makefile文件中生成该文件的相关代码是
		bootfiles = $(call listf_cc,boot)
		$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
	实际命令是：
		gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs 
		-nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc 
		-c boot/bootasm.S -o obj/boot/bootasm.o
	该命令中的主要参数的意义和目的如下：
		-ggdb 生成可以供gdb使用的调试信息；便于使用gdb+qemu进行调试。
		-m32 生成适用于32位环境的代码；因为ucore就是32位的。
		-gstabs 生成调试信息，且格式是stabs的；这是为了能够将调试信息输出到monitor上。
		-nostdinc 不是用标准库；实现操作系统内核的bootloader必须自给自足。		
		-fno-stack-protector 不生成用于检测缓冲区溢出的代码；ucore内核用不到。
		-Ilibs/ libs/是一个目录，这个参数添加了搜索头文件的路径。
		-Os 必要的优化，为了减小代码的大小；主引导扇区只有512字节，其中两个是0X55和0XAA
		所以bootloader大小不能超过510字节。
	bin/sign
	生成sign的makefile相关代码为：
		$(call add_files_host,tools/sign.c,sign,sign)
		$(call create_target_host,sign,sign)
	
	实际命令为：
		gcc -Itools/ -g -Wall -O2 -c tools/sign.c \
		-o obj/sign/tools/sign.o
		gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
	有了必要的文件之后，首先生成bootblock.o，执行的命令为：
		ld -m elf_i386 -nostdlib -N -e start -Ttext 0x7C00 
		obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
	其中关键参数的意义如下：
		-m elf_i386 表示模拟为i386上的连接器。
		-nostdlib  不使用标准库；自给自足。
		-N  设置代码段和数据段均可读写。
		-e start  start是一个entry，目的是指定入口。
		-Ttext  制定代码段开始的位置。
	得到bootblock.o之后使用如下命令将其拷贝到bootblock.out
		objcopy -S -O binary obj/bootblock.o obj/bootblock.out
	其中关键参数的有意义如下：
		-S  移除所有符号和重定位信息。
		-O <bfdname>  指定输出格式。
	使用使用sign工具使用如下的命令处理bootblock.out，生成bootblock。
		bin/sign obj/bootblock.out bin/bootblock
有了基本的工具之后，生成一个有10000个块的文件，每个块默认512字节，用0填充，实际命令如下：
		dd if=/dev/zero of=bin/ucore.img count=10000
然后把bootblock中的内容写到第一个块，实际命令如下：
		dd if=bin/bootblock of=bin/ucore.img conv=notrunc
把kernel中的内容写进接来的块中，实际命令如下：
		dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
支持，我们就能得到ucore.img了。
```
[练习1.2] 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?
```
char buf[512];

buf[510] = 0x55;
buf[511] = 0xAA;
从sign.c的代码的这三行看来，一个硬盘主引导扇区大小是512字节，其中第510和第511位分别用来写入
0x55和0xAA作为标志，前510个字节用来写bootloader。
```

## [练习2] 理解通过make生成执行文件的过程

[练习2.1] 从 CPU 加电后执行的第一条指令开始,单步跟踪 BIOS 的执行。
>在lab1下执行命令
```
	make debug
```
就可以进入gdb的调试界面，在该界面里执行
```
	si
```
就可以进行但不跟踪BIOS的执行了，跟踪期间可以通过
```
	x /2i $pc
```
查看当前eip位置的汇编指令。

[练习2.2] 在初始化位置0x7c00 设置实地址断点,测试断点正常。
>设置断点可以在lab1/tools/gdbinit文件的结尾加上如下代码段：
```
	set architecture i8086
	b *0x7c00
	continue
	x /2i $pc
	set architecture i386
```
其中b *0x7c00用于设置断点，其他语句设置运行环境和显示汇编代码。
做好梗概之后执行
```
	make debug
```
即可看到
```
	Breakpoint 1, 0x00007c00 in ?? ()
	=> 0x7c00:      cli    
	   0x7c01:      cld   
	   0x7c02:      xor    %eax,%eax
	   0x7c04:      mov    %eax,%ds
	   0x7c06:      mov    %eax,%es
	   0x7c08:      mov    %eax,%ss
	   0x7c0a:      in     $0x64,%al
	   0x7c0c:      test   $0x2,%al
	   0x7c0e:      jne    0x7c0a
	   0x7c10:      mov    $0xd1,%al
	   0x7c12:      out    %al,$0x64
	   0x7c14:      in     $0x64,%al...
```
0x7c0c和0x7c0e之间的过程不能显示.

[练习2.3] 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和bootblock.asm进行比较。
得到的结果如下：
```
0x7c00:    cli    
	   0x7c01:      cld   
	   0x7c02:      xor    %eax,%eax
	   0x7c04:      mov    %eax,%ds
	   0x7c06:      mov    %eax,%es
	   0x7c08:      mov    %eax,%ss
	   0x7c0a:      in     $0x64,%al
	   0x7c0c:      test   $0x2,%al
	   0x7c0e:      jne    0x7c0a
	   0x7c10:      mov    $0xd1,%al
	   0x7c12:      out    %al,$0x64
	   0x7c14:      in     $0x64,%al...
```
经比较，与bootasm.S和bootblock.asm中的代码相同
[练习2.4] 自己找一个bootloader或内核中的代码位置,设置断点并进行测试。
设置断点是0x7c16
得到的结果如下：
```
Breakpoint 1, 0x00007c16 in ?? ()
=> 0x7c16:      test   $0x2,%al
   0x7c18:      jne    0x7c14
   0x7c1a:      mov    $0xdf,%al
   0x7c1c:      out    %al,$0x60
   0x7c1e:      lgdtw  0x7c6c
   0x7c23:      mov    %cr0,%eax
   0x7c26:      or     $0x1,%eax
   0x7c2a:      mov    %eax,%cr0...
```
## [练习3] 分析bootloader进入保护模式的过程
>BIOS从硬盘的第一个分区中起始地址为0x7c00的bootloader到内存中，在实模式下开始执行，起始时
%cs=0 %ip=7c00
开始，将几个重要的段寄存器DS、ES、SS置0
```
	start:
	.code16                                             # Assemble for 16-bit mode
	    cli                                             # Disable interrupts
	    cld                                             # String operations increment

	    # Set up the important data segment registers (DS, ES, SS).
	    xorw %ax, %ax                                   # Segment number zero
	    movw %ax, %ds                                   # -> Data Segment
	    movw %ax, %es                                   # -> Extra Segment
	    movw %ax, %ss                                   # -> Stack Segment
```
启用A20：将A20线置于高电位
全部32条地址线可用，可以访问4G的内存空间。
```
	seta20.1:
	    inb $0x64, %al            # Wait for not busy(8042 input buffer empty).
	    testb $0x2, %al
	    jnz seta20.1

	    movb $0xd1, %al           # 0xd1 -> port 0x64
	    outb %al, $0x64           # 0xd1 means: write data to 8042's P2 port

	seta20.2:
	    inb $0x64, %al            # Wait for not busy(8042 input buffer empty).
	    testb $0x2, %al
	    jnz seta20.2

	    movb $0xdf, %al           # 0xdf -> port 0x60
	    outb %al, $0x60           # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
```
初始化GDT表：载入已经存储在引导区中的GDT，不需要重新实现
将cr0寄存器PE位置1就从实模式进入保护模式
```
	    lgdt gdtdesc
	    movl %cr0, %eax
	    orl $CR0_PE_ON, %eax
	    movl %eax, %cr0
```

通过长跳转更新cs的基地址，跳到下一个指令,但在32位的代码段开关处理器会转到32位模式。
```
    ljmp $PROT_MODE_CSEG, $protcseg

.code32                                             # 组装为32位模式
protcseg:
```

设置段寄存器（包括DS、ES、FS、GS），并建立堆栈movw %ax, %ss，此时已经完全进入保护模式
```
	movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
	movw %ax, %ds                                   # -> DS: Data Segment
	movw %ax, %es                                   # -> ES: Extra Segment
	movw %ax, %fs                                   # -> FS
	movw %ax, %gs                                   # -> GS
	movw %ax, %ss                                   # -> SS: Stack Segment
```
调用boot主方法
```
	    call bootmain
```
#[练习4] 分析bootloader加载ELF格式的OS的过程

首先看readsect函数，其作用是从设备的第secno扇区读取数据到dst位置
```
	static void readsect(void *dst, uint32_t secno) {
	    waitdisk();//等待端口空闲
	
	    outb(0x1F2, 1);                         // 设置读取扇区的数目为1
	    outb(0x1F3, secno & 0xFF);
	    outb(0x1F4, (secno >> 8) & 0xFF);
	    outb(0x1F5, (secno >> 16) & 0xFF);
	    outb(0x1waitdisk();F6, ((secno >> 24) & 0xF) | 0xE0);
	    // 上面四条指令联合制定了扇区号，因为每一次都只能传递一
	    // 个byte的值给端口所以32位参数分了四次
	    // 在这4个字节线联合构成的32位参数中
	    //   29-31位强制设为1
	    //   28位(=0)表示访问"Disk 0"
	    //   0-27位是28位的偏移量
	    outb(0x1F7, 0x20);                      // 0x20命令，读取扇区
	
	    waitdisk();

	    insl(0x1F0, dst, SECTSIZE / 4);        
	    //将数据从端口读取到dst，幻数4因为该函数传送时以doubleword为单位，所以传送SECTSIZE/4次
	}
```
readseg简单包装了readsect，可以从设备读取任意长度的内容。
```
static void readseg(uintptr_t va, uint32_t count, uint32_t offset) {
	    uintptr_t end_va = va + count;
	
	    va -= offset % SECTSIZE;//防止超过边界
	
	    uint32_t secno = (offset / SECTSIZE) + 1; 
	    //从字节转换到sector号
	    // 加1因为0扇区被引导占用ELF文件从1扇区开始

	    for (; va < end_va; va += SECTSIZE, secno ++) {
	        readsect((void *)va, secno);
	    }
	    //如果速度比较慢，我们可以一次读大量的sector，事实上写到内存中的比所需的要多
	}
```

在bootmain函数中，
```
	void bootmain(void) {
	    //首先读取ELF的头部
	    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
	
	    //通过储存在头部的幻数判断是否是合法的ELF文件，与ELF_MAGIC不同的话就是不合法的
	    if (ELFHDR->e_magic != ELF_MAGIC) {
	        goto bad;
	    }
	
	    struct proghdr *ph, *eph;
	
	    //ELF头部有描述ELF文件应加载到内存什么位置的描述表，
	    //先将描述表的头地址存在ph
	    //头地址加上ELFHDR的e_phnum就是结束的位置
	    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
	    eph = ph + ELFHDR->e_phnum;
	
	    //按照描述表将ELF文件中数据载入内存对应的位置
	    //从ph到eph的地址范围内
	    for (; ph < eph; ph ++) {
	        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
	    }
	    //ELF文件0x1000位置后面的0xd1ec比特被载入内存0x00100000
	    //ELF文件0xf000位置后面的0x1d20比特被载入内存0x0010e000

	    //根据ELF头部储存的入口信息，找到内核的入口，不返回
	    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
	
	bad:
	    outw(0x8A00, 0x8A00);
	    outw(0x8A00, 0x8E00);
	    while (1);
	}
```
#[练习5] 实现函数调用堆栈跟踪函数
>print_stackframe函数实现如下：
```
void
print_stackframe(void) {
	uint32_t ebp, eip;//初始化两个变量，分别代表ebp和eip中的内容
	ebp = read_ebp();
	eip = read_eip();//获取两个变量的值
	
	for (int i = 0; i < STACKFRAME_DEPTH && ebp != 0; i ++){
		//一直到栈底，且ebp不为0的时候输出以下内容
		uint32_t * args = (uint32_t*) ebp + 2;//函数的参数部分
		cprintf("ebp:0x%08x eip:0x%08x args:", ebp, eip);//输出ebp和eip的值
		for (int j = 0; j < 4; j ++)//输出参数
			 cprintf("0x%08x ", args[j]);
		cprintf("\n");
		print_debuginfo(eip - 1);
		//输出调试信息，因为eip指向的是下一条指令的位置，所以减一
		eip = ((uint32_t *)ebp)[1];
		ebp = ((uint32_t *)ebp)[0];
		//ebp存储的是一个指针指向栈顶部的幀的底部
		//更新eip和ebp的值，分别存在*ebp的前两位
	}

}
```
输出信息如下：
```
	Kernel executable memory footprint: 64KB
	ebp:0x00007b08 eip:0x001009a6 args:0x00010094 0x00000000 0x00007b38 0x00100092 
	    kern/debug/kdebug.c:307: print_stackframe+21
	ebp:0x00007b18 eip:0x00100c95 args:0x00000000 0x00000000 0x00000000 0x00007b88 
	    kern/debug/kmonitor.c:125: mon_backtrace+10
	ebp:0x00007b38 eip:0x00100092 args:0x00000000 0x00007b60 0xffff0000 0x00007b64 
	    kern/init/init.c:48: grade_backtrace2+33
	ebp:0x00007b58 eip:0x001000bb args:0x00000000 0xffff0000 0x00007b84 0x00000029 
	    kern/init/init.c:53: grade_backtrace1+38
	ebp:0x00007b78 eip:0x001000d9 args:0x00000000 0x00100000 0xffff0000 0x0000001d 
	    kern/init/init.c:58: grade_backtrace0+23
	ebp:0x00007b98 eip:0x001000fe args:0x001032fc 0x001032e0 0x0000130a 0x00000000 
	    kern/init/init.c:63: grade_backtrace+34
	ebp:0x00007bc8 eip:0x00100055 args:0x00000000 0x00000000 0x00000000 0x00010094 
	    kern/init/init.c:28: kern_init+84
	ebp:0x00007bf8 eip:0x00007d68 args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8 
	    <unknow>: -- 0x00007d67 --
	++ setup timer interrupts

```
按照函数的运行规则，我们从棧顶向棧底输出，所以最后一个输出的是棧底，也就是第一个调用堆栈的函数，不难
从bootmain.c中看出第一个调用堆栈的函数是bootmain函数，因为堆栈从0x7c00开始，所以ebp应该指向其前
面八位，也就是ebp:0x00007bf8的意义；而eip则是下一条指令的地址。


#[练习6] 完善中断初始化和处理

[练习6.1] 中断描述符表(也可简称为保护模式下的中断向量表)中一个表项占多少字节?其中哪几位代表中断处理
代码的入口?

一个表项占8字节，其中2-3字节是段选择子，0-1字节和6-7字节拼成位移，
两者在一起就是中断处理程序的入口地址。

[练习6.2] 请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。在idt_init函数中,依
次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏,填充idt数组内容。每个中断的入口由tools/vectors
.c生成,使用trap.c中声明的vectors数组即可。

代码及每条语句的含义如下：
```
	void
	idt_init(void) {
	      extern uintptr_t __vectors[];//声明外部变量
	      //所有的中断服务程序都存储在vectors中
	      //__vectors[]出现在kern/trap/vector.S中
	      //在执行了make之后就可以基于tools/vector.c生成vector.S
	      int i;
	      for (i = 0; i < sizeof(idt) / sizeof(struct gatedesc); i ++)
	      	SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);
	      
	      SETGATE(idt[T_SWITCH_TOK], 0, GD_KTEXT, __vectors[T_SWITCH_TOK], DPL_USER);
	      //启用中断描述符表中的每一个条目，以便于从用户态转到内核态
	      
	      lidt(&idt_pd);
	      //使用lidt指令告诉CPU中断描述符表的位置
	}
```

[练习6.3] 请编程完善trap.c中的中断处理函数trap,在对时钟中断进行处理的部分填写trap函数中处理时钟中断
的部分,使操作系统每遇到100次时钟中断后,调用print_ticks子程序,向屏幕上打印一行文字”100	ticks”。

只需要进行计数就可以了，代码如下：
```
	void
	trap(struct trapframe *tf) {
	    // dispatch based on what type of trap occurred
	    trap_dispatch(tf);
	    interrupt_num ++;
	    if(interrupt_num >= 99){//满一百输出一次
		print_ticks();
		iterrput_num = 0;//重新置0
	    }
	}
```
在trap_dispatch中的case IRQ_OFFSET + IRQ_TIMER:需要改成
        ticks ++;
        if (ticks % TICK_NUM == 0) {
            print_ticks();
        }
	break；










