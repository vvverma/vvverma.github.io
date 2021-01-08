[Back to Main Page](README.md)

Softirqs 
============

* TOC
{:toc}


Softirqs are bottom halves that run at a high priority but with hardware interrupts enabled

Implementation: kernel/softirq.c

Header File: <linux/softirq.h>

Data structures
---------------

Softirqs are represented by the softirq_action structure.

struct softirq_action
{
        void    (*action)(struct softirq_action *);
};

A 10 entry array of this structure is declared in kernel/softirq.c

static struct softirq_action softirq_vec[NR_SOFTIRQS];

	two for tasklet processing (HI_SOFTIRQ and TASKLET_SOFTIRQ),
	two for send and receive operations in networking (NET_TX_SOFTIRQ and NET_RX_SOFTIRQ),
	two for the block layer (asynchronous request completions),
	two for timers, and 
	one each for the scheduler and 
	read-copy-update processing

From include/linux/interrupt.h
enum
{
        HI_SOFTIRQ=0,
        TIMER_SOFTIRQ,
        NET_TX_SOFTIRQ,
        NET_RX_SOFTIRQ,
        BLOCK_SOFTIRQ,
        IRQ_POLL_SOFTIRQ,
        TASKLET_SOFTIRQ,
        SCHED_SOFTIRQ,
        HRTIMER_SOFTIRQ, /* Unused, but kept as tools rely on the
                            numbering. Sigh! */
        RCU_SOFTIRQ,    /* Preferable RCU should always be the last softirq */

        NR_SOFTIRQS
};


The number of registered softirqs is statically determined and cannot be changed dynamically.


Preeemption
-------------
A softirq never preempts another softirq . The only event that can preempt a softirq is interrupt handler.

Another softirq even the same one can run on another processor.
/proc/softirqs
================

Shows Per CPU statistics

Implementation: fs/proc/softirqs.c

$ watch -n1 grep RX /proc/softirqs



Important Points related to softirqs
-------------------------------------

1. Compile Time:
	Declared at compile time in an enumerator
	Not suitable for linux kernel modules

2. Execution:
	Executed as early as possible
		After return of a top handler and before return to a system call
	This is achieved by giving a high priority to the executed softirq handlers

3. Parallel:
	Softirqs can run in parallel
	Each processor has its own softirq bitmap
	One softirq cannot be scheduled twice on the same processor
	One softirq may run in parallel on other

4. Priority:
	Kernel iterates over the softirq bitmap, least significant bit (LSB) first, and execute the associated
	softirq handlers
	
ksoftirqd
============

Softirqs are executed as long as the processor-local softirq bitmap is set.

Since softirqs are bottom halves and thus remain interruptible during execution, 

the system can find itself in a state where it does nothing else than 
	serving interrupts and 
	softirqs

incoming interrupts may schedule softirqs what leads to another iteration over the bitmap.

Such processor-time monopolization by softirqs is acceptable under high workloads (e.g., high IO or network traffic), but it is generally undesirable for a longer period of time since (user) processes cannot be executed.

Solution to above problem by kernel
======================================

After the tenth iteration(MAX_SOFTIRQ_RESTART) over the softirq bitmap, the kernel schedules the so-called ksoftirqd kernel thread, which takes control over the execution of softirqs.

Each processor has its own kernel thread called ksoftirqd/n, where n is the number of the processor

This processor-local kernel thread then executes softirqs as long as any bit in the softirq bitmap is set.

The aforementioned processor-monopolization is thus avoided by deferring softirq execution into process context (i.e., kernel thread), so that the ksoftirqd can be preempted by any other (user) process.

 ps -ef | grep ksoftirqd/

The spawn_ksoftirqd function starts these threads.

It is called early in the boot process.

static __init int spawn_ksoftirqd(void)
{
        cpuhp_setup_state_nocalls(CPUHP_SOFTIRQ_DEAD, "softirq:dead", NULL,
                                  takeover_tasklets);
        BUG_ON(smpboot_register_percpu_thread(&softirq_threads));

        return 0;
}
early_initcall(spawn_ksoftirqd);

File: kernel/softirq.c

Each ksoftirqd/n kernel thread runs the run_ksoftirqd()

static void run_ksoftirqd(unsigned int cpu)
{
        local_irq_disable();
        if (local_softirq_pending()) {
                /*
                 * We can safely run softirq on inline stack, as we are not deep
                 * in the task stack here.
                 */
                __do_softirq();
                local_irq_enable();
                cond_resched();
                return;
        }
        local_irq_enable();
}


local_softirq_pending()
-----------------------

It is 32-bit mask of pending softirqs
When are pending softirqs run?
-------------------------------

Pending softirq handlers are checked and executed at various points in the kernel code.

	a) After the completion of hard interrupt handlers with IRQ Lines Enabled
		do_IRQ() function finishes handling an I/O interrupt and invokes the irq_exit()

	b) call to functions like local_bh_enable() or spin_unlock_bh()

	c) when one of the special ksoftirqd/n kernel threads is awakened
in_softirq
-----------------

You can tell you are in a softirq (or tasklet) using the in_softirq() macro
Disabling/Enabling Softirqs
==============================

If a softirq shares data with user context, you have two problems.

1) the current user context can be interrupted by a softirq

2) the critical region could be entered from another CPU

Solution to  first problem
---------------------------

void local_bh_disable() 	Disable softirq and tasklet processing on the local processor

void local_bh_enable()		Enable softirq and tasklet processing on the local processor

The calls can be nested only the final call to local_bh_enable() actually enables bottom halves.
softirq methods
===================

Registering softirq handlers:
-------------------------

Software interrupts must be registered before the kernel can execute them.

open_softirq is used for associating the softirq instance with the corresponding bottom halve routine.

void open_softirq(int nr, void (*action)(struct softirq_action *))
{
        softirq_vec[nr].action = action;
}

It is being called for example from networking subsystem.

net/core/dev.c:
---------------
        open_softirq(NET_TX_SOFTIRQ, net_tx_action);
        open_softirq(NET_RX_SOFTIRQ, net_rx_action);

Execution of softirq:
----------------------

The kernel maintains a per-CPU bitmask indicating which softirqs need processing at any given time
irq_stat[smp_processor_id].__softirq_pending.

Drivers can signal the execution of soft irq handlers using a function raise_softirq().
This function takes the index of the softirq as argument.

void raise_softirq(unsigned int nr)
{
        unsigned long flags;

        local_irq_save(flags);
        raise_softirq_irqoff(nr);
        local_irq_restore(flags);
}


local_irq_save 		--> Disables interrupts on the current processor where code is running
raise_softirq_irqoff 	--> sets the corresponding bit in the local CPUs softirq bitmask to mark the specified softirq as pending
local_irq_restore	--> Enables the interrupts

raise_softirq_irqoff if executed in non-interrupt context, will invoke wakeup_softirqd(), to wake up, if necessary the ksoftirqd kernel thread of that local CPU

What is the benefit of per-CPU Bitmask?
========================================

By using a processors specific bitmap, the kernel ensures that several softIRQs — even identical ones — can be executed on different CPUs at the same time.

Executing Softirqs
=====================

The actual execution of softirqs is managed by do_softirq()

Implementation : kernel/softirq.c

do_softirq() will call __do_softirq(), if any bit in the local softirq bit mask is set.

__do_softirq() then iterates over the softirq bit mask (least signicant bit) and invokes scheduled softirq handlers.


Locking Between User Context and Softirqs
==========================================

spin_lock_bh()	Disables softirqs on the CPU and then grabs the lock

spin_unlock_bh() Release lock and enable softirqs


Creating a new softirq
-----------------------

You declare softirqs statically at compile time via an enum in <linux/interrupt.h>.

Creating a new softirq includes adding a new entry to this enum.

The index is used by the kernel as priority.

Softirqs with the lowest numerical priority execute before those with a higher numerical priority

Insert the new entry depending on the priority you want to give it.

Registering your handler
------------------------

Soft irq is registered at runtime via open_softirq().

It takes two parameters:

	a) Index
	b) Handler Function.

Raising your softirq
---------------------

To mark it pending, so it is run at the next invocation of do_softirq(), call raise_softirq()

Softirqs are most often raised from within interrupt handlers.



Other Details
-----------------
The softirq handlers run with interrupts enabled and cannot sleep.

While a handler runs, softirqs on the current processor are disabled.

Another processor, can however execute another softirq.

If the same softirq is raised again while it is executing, another processor can run in it simultaneously.

This means that any shared data even global data used only within the soft irq handler needs proper locking.

most softirq handlers resort to per-processor data (data unique to each processor and thus not requiring locking) and other tricks to avoid explicit locking and provide excellent scalability.
