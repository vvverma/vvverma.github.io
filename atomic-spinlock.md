[Back to Main Page](README.md)

Atomic Operations
==================

* TOC
{:toc}

Several assembly language instructions are of type “read-modify-write”
	they access a memory location twice,
		 the first time to read the old value and 
		 the second time to write a new value.

Suppose that two kernel control paths running on two CPUs try to “read-modify-write” the same memory location at the same time by executing nonatomic operations.

	At first, both CPUs try to read the same location, but the memory arbiter (a hardware circuit that serializes accesses to the RAM chips) steps in to grant access to one of them and delay the other.

	However, when the first read operation has completed, the delayed CPU reads exactly the same (old) value from the memory location.

	Both CPUs then try to write the same (new) value to the memory location; again, the bus memory access is serialized by the memory arbiter, and eventually both write operations succeed.

	However, the global result is incorrect because both CPUs write the same (new) value. Thus, the two interleaving “read-modify-write” operations act as a single one.

	Kernel Thread1				Kernel Thread2	
	------------------------------------------------------

	read i (5)
						read i(5)

	increment i(5 -> 6)	
		
						increment i (5 -> 6)

	write i(6)

						write i(6)
Non atomic bitwise operations
===============================

Conveniently, nonatomic versions of all the bitwise functions are also provided.

They behave identically to their atomic siblings, except they do not guarantee atomicity, and their names are prefixed with double underscores.

For example, the nonatomic form of test_bit() is __test_bit().

If you do not require atomicity (say, for example, because a lock already protects your data), these variants of the bitwise functions might be faster.
Atomic Operations
==================
The easiest way to prevent race conditions due to “read-modify-write” instructions is by ensuring that such operations are atomic at the chip level. 

Every such operation must be executed in a single instruction without being interrupted in the middle and avoiding accesses to the same memory location by other CPUs.

 Most CPU instruction set architectures define instruction opcodes that can perform atomic read-modify-write operations on a memory location. 

In general, special lock instructions are used to prevent the other processors in the system from working until the current processor has completed the next action


atomic_t
=========
When you write C code, you cannot guarantee that the compiler will use an atomic instruction for an operation like a = a + 1 or even for a + +.

Thus, the Linux kernel provides a special atomic_t type (an atomically accessible counter) and some special functions and macros that act on atomic_t variables

Header File: <asm/atomic.h>
typedef struct { volatile int counter; } atomic_t;

Why a new user defined data type atomic_t is needed?
======================================================

Because the atomic data types are ultimately implemented with normal C types, the kernel
encapsulates standard variables in a structure that can no longer be processed with normal operators
such as ++.


What happens to atomic variables when the kernel is compiled without SMP Support?
==================================================================================

it works the same way as for normal variables (only atomic_t encapsulation is observed) because there is no interference from other processors


atomic_t
============

Initialization:
===================

atomic_t i;  //define i

atomic_t i = ATOMIC_INIT(1); //define i and initialize it to 1


Increment/Decrement
====================

void atomic_inc(atomic_t *i);  //Add 1 to *i
void atomic_dec(atomic_t *i);  //Subtract 1 from *i

Set/Read
========

void atomic_set(atomic_t *i, int j); //Atomically set counter i to value specified in j
int atomic_read(atomic_t *i); //Read value of the atomic counter i

Add/Sub
===========
void atomic_add(int val, atomic_t *i); //Atomically add val to atomic counter i
void atomic_sub(int val, atomic_t *i); //Atomically subtract val from atomic counter i




Common use of atomic operations
===============================

	A common use of the atomic integer operations is to implement counters
	Protecting a sole counter with a complex locking scheme is overkill, so instead developers use 
	atomic_inc() and atomic_dec(), which are much lighter in weight.
Atomic Operation and test
=========================

int atomic_dec_and_test(atomic_t *i); //atomic Subtract 1 from *i and return 1 if the result is zero; 0 otherwise

int atomic_inc_and_test(atomic_t *i); //atomic Add 1 to *i and return 1 if the result is zero; 0 otherwise

// Atomically subtract val from *i and return 1 if the result is zero; otherwise 0
int atomic_sub_and_test(int val, atomic_t *i); 

//Atomically add val to *i and return 1 if the result is negative; otherwise 0
int atomic_add_negative(int val, atomic_t *i);


Atomic Add/Subtract and return
==============================

int atomic_add_return(int val, atomic_t *i);// Atomically add val to *i and return the result.
int atomic_sub_return(int val, atomic_t *i); // Atomically subtract val from *i and return the result.
int atomic_inc_return(atomic_t *i);// Atomically increment *i by one and return the result.
int atomic_dec_return(atomic_t *i);// Atomically decrement *i by one and return the result.


More atomic operations
========================
//Atomically adds val to i and return pre-addition value at i
int atomic_fetch_add(int val, atomic_t *i);


//Atomically subtracts val from i, and return pre-subtract value at i
int atomic_fetch_sub(int val, atomic_t *v);


//Reads the value at location i, and checks if it is equal to old; if true, swaps value at v with new, and always returns value read at i

int atomic_cmpxchg(atomic_t *i, int old, int new);

//Swaps the oldvalue stored at location i with new, and returns old value i
int atomic_xchg(atomic_t *i, int new);

64-bit Atomic Operations
===========================

Many processor architectures have no 64-bit atomic instructions, but we need atomic64_t in order to support the perf_counter subsystem.

This adds an implementation of 64-bit atomic operations using hashed spinlocks to provide atomicity.

typedef struct {
	long long counter;
} atomic64_t;

These functions have the naming convention atomic64_*()
Atomic Bitwise Operations
=============================

In addition to atomic integer operations, the kernel also provides a family of functions that operate at the bit level.

Header File: <asm/bitops.h>

These functions operate on generic pointer. There is no equivalent of the atomic integer atomic_t.

Arguments:
===========
	1. Bit Number: 0 - 31 for 32 bit machines and 0 - 63 for 64-bit machines
	2. Pointer with valid address

//Atomically set the bit nr in location starting from addr
void set_bit(int nr, volatile unsigned long *addr);

//Atomically clear the nr-th bit starting from addr.
void clear_bit(int nr, volatile unsigned long *addr);

//Atomically flip the value of the nr-th bit starting from addr.
void change_bit(int nr, volatile unsigned long *addr);
Atomic bit operations with return value
========================================

//Atomically set the bit nr in the location starting from p, and return old value at the nrth bit
int test_and_set_bit(unsigned int nr, volatile unsigned long *p);

//Atomically clear the bit nr in the location starting from p, and return old value at the nrthbit
int test_and_clear_bit(unsigned int nr, volatile unsigned long *p);

//Atomically flip the bit nr in the location starting from p, and return old value at the nrth bit
int test_and_change_bit(unsigned int nr, volatile unsigned long *p);
Problem with atomic instructions:
=================================

        Can only work with CPU word and double word size.
        Cannot work with shared data structures of custom size.


In real life, critical regions can be more than one line. And these code paths such execute atomically to avoid race condition.

To ensure atomicity of such code blocks locks are used.



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
