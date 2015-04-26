#Lab4实验报告
计22，张天成，2012011267

<br>
##练习1

###设计实现过程

alloc_proc函数的实现见代码，代码中笔者实现的部分说明如下：

	proc->state = PROC_UNINIT;	//初始化进程状态为PROC_UNINIT	proc->pid = -1;				//初始化进程id为－1	proc->runs = 0;				//初始化进程运行次数为0	proc->kstack = 0;			//初始化内核栈为空	proc->need_resched = 0;		//初始化进程调度状态为不需要	proc->parent = NULL;		//初始化进程父进程为空	proc->mm = NULL;			//初始化进程存储结构为空	memset(&(proc->context), 0, sizeof(struct context));		//初始化进程的context部分为全零以便后续填充	proc->tf = NULL;			//初始化进程的中断帧为空	proc->cr3 = boot_cr3;		//初始化进程的PDT基址为内核PDT基址	proc->flags = 0;			//初始化进程的标志为0	memset(proc->name, 0, PROC_NAME_LEN);						//初始化进程的名称域为全零以便后续填充

###思考题回答
proc_struct中struct context context的作用是保存进程的上下文（即各寄存器的值），以在进程切换时恢复各寄存器的值，使进程继续正常执行。
proc_struct中struct trapframe tf*的作用是保存进程的中断帧，在进程从用户空间切换至内核空间时，中断帧记录进程在被中断前的状态，以便在从内核空间跳回用户空间时恢复GS、ES、DS等寄存器的值，使进程继续正常运行。
##练习2
###设计实现过程
do_fork函数的实现见代码，代码中笔者实现的部分说明如下：
	// 1. 调用alloc_proc函数分配一个进程控制块    proc = alloc_proc();    if(proc == NULL)    	goto fork_out;    // 2. 调用setup_kstack函数为子进程分配内核栈    if(setup_kstack(proc) != 0)    	goto bad_fork_cleanup_proc;    // 3. 调用copy_mm函数以根据clone_flags复制或共享内存空间    if(copy_mm(clone_flags, proc) != 0)    	goto bad_fork_cleanup_kstack;    // 4. 调用copy_thread函数以设置好子进程的中断帧和上下文    copy_thread(proc, stack, tf);    // 5. 屏蔽中断，设置新进程的进程id，将新的进程控制块插入hash_list和proc_list中，完成后再开启中断    bool intr_flag;    local_intr_save(intr_flag);    {		proc->pid = get_pid();		hash_proc(proc);		list_add(&proc_list, &(proc->list_link));		nr_process ++;	}	local_intr_restore(intr_flag);    // 6. 调用wakup_proc将新进程的状态设置为PROC_RUNNABLE    wakeup_proc(proc);    // 7. 将返回值设置为子进程的id    ret = proc->pid;
###思考题回答

ucore实现了对每个新fork得到的线程分配一个唯一的id。在do_fork函数中，为新进程设置的id由get_pid函数得到，而在get_pid函数中，ucore检查目前的所有进程，找到最小的未被已有线程占用且不超过最大允许id值的id返回，故返回的id是唯一的，则新fork出的线程的id也是唯一的。

##练习3

###对proc_run函数的分析

对proc_run函数的代码分析如下：

	void	proc_run(struct proc_struct *proc) {    	
    	//若proc不为当前线程，则执行切换    	if (proc != current) {    	        	//intr_flag保存中断标志        	bool intr_flag;        	
			//prev, next分别为当前（切换前）、下一（切换后）进程        	struct proc_struct *prev = current, *next = proc;
        	//屏蔽中断        	local_intr_save(intr_flag);        	{            	//进程控制块切换            	current = proc;            	
				//内核栈切换            	load_esp0(next->kstack + KSTACKSIZE);            	
				//页目录表切换            	lcr3(next->cr3);            	            	//上下文（寄存器值等）切换            	switch_to(&(prev->context), &(next->context));        	}
			//开启中断        	local_intr_restore(intr_flag);        	    	}    	//切换完成	}
###思考题回答
1. 在本实验的执行过程中，创建且运行了几个内核线程？

	两个。通过查看proc_init函数知，本实验的执行过程中创建且运行了idleproc和initproc两个内核线程，其中idleproc由init_proc函数直接创建，initproc由kernel_thread函数以fork的方式创建。2. 语句local_intr_save(intr_flag);....local_intr_restore(intr_flag);在这里有何作用？请说明理由。
	进程切换的过程中需要保证对current指针操作的原子性，故目前需要屏蔽中断。local_intr_save(intr_flag)在进程切换前起到保存当前中断标志并屏蔽中断的作用；local_intr_restore(intr_flag)起到恢复屏蔽中断前的中断标志并开启中断的作用。