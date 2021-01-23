[Back to Main Page](README.md)

Spinlocks
==============

The most common lock in the Linux kernel is the spin lock.

Spinlocks are used to protect short code sections that comprise just a few C statements and are therefore
quickly executed and exited.

A spin lock is a lock that can be held by at most one thread of execution.

When the thread tries to acquire lock which is already held?
--------------------------------------------------------------
        The thread busy loops/spins waiting for the lock to become available.

When the thread tries to acquire lock which is available?
--------------------------------------------------------------
        The thread acquires the lock and continues.

Important Points with Spinlocks
===============================

1. If a lock is acquired but no longer released, the system is rendered unusable.

	All processors— including the one that acquired the lock — sooner or later arrive at a point where they must enter the critical region.

	They go into the endless loop to wait for lock release, but this never happens and deadlocks.

2. On no account should spinlocks be acquired for a longer period because all processors waiting for lock release are no longer available for other productive tasks.

3. Code that is protected by spinlocks must not go to sleep.
	it must also be ensured that none of the functions that are called inside a spinlocked region can go to sleep!
	Ex: kmalloc with GFP_KERNEL.


	
Why was there no error?
=======================
	Because no other process tried to take the same spinlock.

For example, 
	thread A acquires a spin lock.
	Thread A will not be preempted until it releases the lock
	 But if thread A sleeps while holding the lock
	thread B could be scheduled to run since the sleep function will invoke the scheduler. 
	And thread B could acquire the same lock as well.
	Deadlock
	


How are spinlocks implemented
=============================

Simple Implementation
----------------------
	
	A spinlock is a mutual exclusion device that can have only two values:
		Locked
		Unlocked

	It is usually implemented as a single bit in an integer value.

	Code wishing to take out a particular lock tests the relevant bit.
		If the lock is available, the "locked" bit is set and the code continues into the critical section		   If, instead, the lock has been taken by somebody else, the code goes into a tight loop where it repeatedly checks the lock until it becomes available.
		 This loop is the "spin" part of a spinlock.


Real Implementation
--------------------

The "test and set" operation must be done in an atomic manner so that only one thread can obtain the lock, even if several are spinning at any given time. 

Spinlocks are built on top of hardware-specific atomic instructions.

The actual implementation of spinlock is different for each architecture the linux supports.




Spinlock Methods
=====================

Header File: <linux/spinlock.h>
	

Data Structure: spinlock_t


Methods
------------

DEFINE_SPINLOCK(my_lock);   == spinlock_t my_lock = __SPIN_LOCK_UNLOCKED(my_lock);

From <linux/spinlock_types.h>:
#define DEFINE_SPINLOCK(x)      spinlock_t x = __SPIN_LOCK_UNLOCKED(x)

//To lock a spin lock
void spin_lock(spinlock_t *lock);

//To unlock a spin lock
void spin_unlock(spinlock_t *lock);
To initialize spin lock at run time.

void spin_lock_init(spinlock_t *lock);
Will spin lock exist on uniprocessor machines?
==============================================

When CONFIG_PREEMPT is not set/kernel preemption disabled
===========================================================

spinlocks are defined as empty operations because critical sections cannot be entered by several CPUs at the same time.

When CONFIG_PREEMPT is set
===========================

	spin_lock  = preempt_disable
	spin_unlock = preempt_enable
spin_trylock()
---------------

int spin_trylock(spinlock_t *lock);

Tries to acquire given lock;

	If not available, returns zero.

	If available, it obtains the lock and returns nonzero
Is the kernel preemption disabled when the spinlock is acquired?
=================================================================
Is the kernel preemption disabled when the spinlock is acquired?
=================================================================

Any time kernel code holds a spinlock, preemption is disabled on the relevant processor

Lock/unlock methods disable/enable kernel preemption
Can i use spinlock when the resource is shared between kernel control path in process context vs interrupt context
==================================================================================================================
Can i use spinlock when the resource is shared between kernel control path in process context vs interrupt context
==================================================================================================================

	1. Your driver is executing and has taken a lock.

	2. Device the driver is handling issues a interrupt.

	3. Interrupt handler also obtains the same lock.

	
Problem: Happens when the interrupt handler runs in the same processor which driver code with lock is running

	Interrupt handler will spin forever, as the non interrupt code will not be able to run to release the lock


Solution:

	Disable interrupts before acquiring the spin lock

	Enable them back after releasing the spin lock.

The kernel provides an interface that conveniently disables interrupts and acquires the lock.

DEFINE_SPINLOCK(my_lock);
unsigned long flags;

spin_lock_irqsave(&my_lock, flags);
/* critical region ... */
spin_unlock_irqrestore(&my_lock, flags);

Why additional argument of flags is needed
============================================

What if the interrupts were disabled before you acquire a spinlock, if we don't have flags, we will enable them after unlocking.

spin_lock_irqsave()saves the current state of interrupts, disables them locally, and then obtains the given lock.

Conversely, spin_unlock_irqrestore() unlocks the given lock and returns interrupts to their previous state.


If you always know before the fact that interrupts are initially enabled, there is no need to restore their previous state

DEFINE_SPINLOCK(mr_lock);
spin_lock_irq(&mr_lock);
/* critical section ... */
spin_unlock_irq(&mr_lock);

Use of spin_lock_irq() is not recommended, as it is hard to ensure interrupts are always enabled in any kernel code path

	
What happens if i acquire a lock which is already held by the same CPU?
Spin locks are not recursive
===============================

Unlike spin lock implementations in other operating systems and threading libraries, the Linux kernel’s spin locks are not recursive

This means that if you attempt to acquire a lock you already hold, you will spin, waiting for yourself to release the lock.

But because you are busy spinning, you will never release the lock and you will deadlock.
