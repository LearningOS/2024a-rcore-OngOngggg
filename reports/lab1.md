# Lab1

## 功能实现

​	调用sys_task_info函数时，当前任务处于运行态，所以只需要关心调度次数即可

​	从任务的TCB头重获取任务的调用次数，设置`syscall_times`为一个统计每种系统调用次数的数组，在系统调用时提供`syscall_to_increase`函数以统计+1，同时设置任务`syscall_times`对应的get函数

​	同样在TCB中设置开始时间，初始为0，在任务第一次调度`run_first_task`/`run_first_task` 设置为系统已执行时间，在调用函数时他与系统执行时间的差即为任务执行时间（不过为什么直接get_time_ms也能过）



## 简答作业

1. 正确进入 U 态后，程序的特征还应有：使用 S 态特权指令，访问 S 态寄存器后会报错。 请同学们可以自行测试这些内容（运行 [三个 bad 测例 (ch2b_bad_*.rs)](https://github.com/LearningOS/rCore-Tutorial-Test-2024A/tree/master/src/bin) ）， 描述程序出错行为，同时注意注明你使用的 sbi 及其版本。

	version： `RustSBI version 0.3.0-alpha.2, adapting to RISC-V SBI v1.0.0`

	ch2b_bad_address：日志`[kernel] PageFault in application, bad addr = 0x0, bad instruction = 0x804803a4, kernel killed it.`在0x0地址写入数据访问未映射的地址，引发非法访问

	ch2b_bad_instructions：日志`IllegalInstruction in application, kernel killed it.`在用户态使用sret非法

	ch2b_bad_register：日志`IllegalInstruction in application, kernel killed it.`用户态获取sstatus也是不行的

2. 深入理解 [trap.S](https://github.com/LearningOS/rCore-Camp-Code-2024A/blob/ch3/os/src/trap/trap.S) 中两个函数 `__alltraps` 和 `__restore` 的作用，并回答如下问题:

	1. L40：刚进入 `__restore` 时，`a0` 代表了什么值。请指出 `__restore` 的两种使用情景。

		对应指向TrapContext的指针

		`__restore`用于执行完`__alltraps`异常处理后的返回，以恢复之前的寄存器、栈指针状态等；或者在时间片轮转时恢复被挂起任务的上下文

	2. L43-L48：这几行汇编代码特殊处理了哪些寄存器？这些寄存器的的值对于进入用户态有何意义？请分别解释。

		```
		ld t0, 32*8(sp)
		ld t1, 33*8(sp)
		ld t2, 2*8(sp)
		csrw sstatus, t0
		csrw sepc, t1
		csrw sscratch, t2
		```

		t0-----sstatus 给出 Trap 发生之前 CPU 处在哪个特权级（S/U）等信息，前后执行环境一致
		t1-----sepc 记录发生异常的指令地址，保证程序返回后能找到对应位置
		t2-----sscratch 用于保存用户态指针，恢复现场用的

	3. L50-L56：为何跳过了 `x2` 和 `x4`？

		```
		ld x1, 1*8(sp)
		ld x3, 3*8(sp)
		.set n, 5
		.rept 27
		   LOAD_GP %n
		   .set n, n+1
		.endr
		```

		x2对应的就是sp指针，一般在函数最开头或结尾维护

		x4是多线程用的？？？感觉这里没用到吧

	4. L60：该指令之后，`sp` 和 `sscratch` 中的值分别有什么意义？

		```
		csrrw sp, sscratch, sp
		```

		sp对应用户态的栈指针，而sscratch对应内核栈

	5. `__restore`：中发生状态切换在哪一条指令？为何该指令执行之后会进入用户态？

		最后一条指令`sret`；执行对应命令后，内核栈中的sstatus sepc 等信息被拿出来返回到用户态，对应执行`csrw sstatus t0`等指令

	6. L13：该指令之后，`sp` 和 `sscratch` 中的值分别有什么意义？

		```
		csrrw sp, sscratch, sp
		```

		sp到内核态，而sscratch到用户态

	7. 从 U 态进入 S 态是哪一条指令发生的？

​			  `“在 Trap 触发的一瞬间， CPU 会切换到 S 特权级并跳转到 stvec 所指示的位置”`

​			然后在Trap发生后，对应执行Trap.S中的两个函数

​				