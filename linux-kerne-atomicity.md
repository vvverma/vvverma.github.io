# Linux Kernel Concept : Atomicity


* TOC
{:toc}


## Motivation for Atomicity

Several assembly language instructions are of type “read-modify-write” , these access a memory location twice i.e the first time to read the value present and  the second time to write a update value.

Suppose that two kernel control paths running on two CPUs try to “read-modify-write” the same memory location at the same time by executing nonatomic operations. At first, both CPUs try to read the same location, but the memory arbiter (a hardware circuit that serializes accesses to the RAM chips) steps in to grant access to one of them and delay the other. However, when the first read operation has completed, the delayed CPU reads exactly the same (old) value from the memory location. Both CPUs then try to write the same (new) value to the memory location; again, the bus memory access is serialized by the memory arbiter, and eventually both write operations succeed. However, the global result is incorrect because both CPUs write the same (new) value. Thus, the two interleaving “read-modify-write” operations act as a single one.
|Kernel Thread-1| Kernel Thread-2|	
|----------|----------------|
|read i (5)||
|| read i(5)|
|increment i(5 -> 6)||
||increment i (5 -> 6)|
|write i(6)||
||write i(6)|

In the scenrio above, when both Kernel Thread-1 and Kernel Thread-2 tries to access the same memory location to read, the memory bus arbirator provides access to one of them. As seen above Kernel Thread-1 gets the access and reads the value of i as 5 and then Kernel Thread-2 also reads the old value of 5 (which should be 6 post Thread-1 updates). The value in i should have been 7, since there no atomic exection read-modify-write. Hence, this gives us the motivation to have instructions which should be atomic in nature. 

The easiest way to prevent race conditions (as seen above) due to “read-modify-write” instructions is by ensuring that such operations are atomic at the chip level.  Every such operation must be executed in a single instruction without being interrupted in the middle and avoiding accesses to the same memory location by other CPUs. Most CPU instruction set architectures define instruction opcodes that can perform atomic read-modify-write operations on a memory location. In general, special lock instructions are used to prevent the other processors in the system from working until the current processor has completed the next action.

Conveniently, nonatomic versions of all the bitwise functions are also provided. They behave identically to their atomic siblings, except they do not guarantee atomicity, and their names are prefixed with double underscores. 

``` 
test_bit() is __test_bit()
 ```

If you do not require atomicity (say, for example, because a lock already protects your data), these variants of the bitwise functions might be faster.

In the upcoming section we will see how we use use or write code with gaurantees us the atomicity


## The Datatype :  **atomic_t**

When you write C code, you cannot guarantee that the compiler will use an atomic instruction for an operation like a = a + 1 or even for a + +. Thus, the Linux kernel provides a special atomic_t type (an atomically accessible counter) and some special functions and macros that act on atomic_t variables

Header File: *\<asm/atomic.h>*

```
typedef struct { 
        volatile int counter;
} 
atomic_t;

```
As we see above, atomic_t seems as userdefined datype. The user defined atomic_t is needed as the atomic data types are ultimately implemented with normal C types, the kernel encapsulates standard variables in a structure that can no longer be processed with normal operators such as ++.

Atomicity is needed or most effective when the system/Hardware has SMP support. Suppose if we compile Linux Kernel without SMP Support then it works the same way as for normal variables (only atomic_t encapsulation is observed) because there is no interference from other processors


## Initialization and Types of Operation on  **atomic_t** datatype


### Intialization of atomic_t user datatype *i*

```
/* define i */
atomic_t i;

/*define i and initialize it to 1*/
atomic_t i = ATOMIC_INIT(1);
```

### Increment/Decrement of atomic_t user datatype *i*

```
/*  Add 1 to *i  */
void atomic_inc(atomic_t *i);

/* Subtract 1 from *i  */
void atomic_dec(atomic_t *i);  
```

### Atomic Set/Read of atomic_t user datatype *i*

```
/* Atomically set counter i to value specified in j */
void atomic_set(atomic_t *i, int j);

/*Read value of the atomic counter i */
int atomic_read(atomic_t *i);   
        
```

### Atomic Add/Sub of atomic_t user datatype *i*

```
/* Atomically add val to atomic counter i  */
void atomic_add(int val, atomic_t *i);

/* Atomically subtract val from atomic counter i */
void atomic_sub(int val, atomic_t *i);
```

### Atomic Operation and test of atomic_t user datatype *i*

```
/* Atomic Subtract 1 from *i and return 1 if the result is zero; 0 otherwise */
int atomic_dec_and_test(atomic_t *i);

/* Atomic Add 1 to *i and return 1 if the result is zero; 0 otherwise */
int atomic_inc_and_test(atomic_t *i);

/* Atomic Subtract val from *i and return 1 if the result is zero; otherwise 0 */
int atomic_sub_and_test(int val, atomic_t *i); 

/* Atomic add val to *i and return 1 if the result is negative; otherwise 0 */
int atomic_add_negative(int val, atomic_t *i);

```

### Atomic Add/Subtract and return of atomic_t user datatype *i*

```
 /*Atomically add val to *i and return the result  */
int atomic_add_return(int val, atomic_t *i);

/*Atomically subtract val from *i and return the result */
int atomic_sub_return(int val, atomic_t *i);


/*Atomically increment *i by one and return the result */
int atomic_inc_return(atomic_t *i);

 /*Atomically decrement *i by one and return the result */
int atomic_dec_return(atomic_t *i); 
```

### Additonal atomic operations of atomic_t user datatype *i*

```
/* Atomically adds val to i and return pre-addition value at i */
int atomic_fetch_add(int val, atomic_t *i);

/*Atomically subtracts val from i, and return pre-subtract value at i */
int atomic_fetch_sub(int val, atomic_t *v);

/*Reads the value at location i, and checks if it is equal to old ;
if true, swaps value at v with new and always returns value read at i */
int atomic_cmpxchg(atomic_t *i, int old, int new);

/* Swaps the old value stored at location i with new, and returns
old value *i */
int atomic_xchg(atomic_t *i, int new);

```

## 64-bit Atomic Operations

Many processor architectures have no 64-bit atomic instructions, but we need atomic64_t in order to support the perf_counter subsystem. As seen below, adds an implementation of 64-bit atomic operations using hashed spinlocks to provide atomicity. 64-bit atomic operation supported functions have the naming convention atomic64_*()

```
typedef struct {
	long long counter;
} atomic64_t;

```

### Atomic Bitwise Operations using atomic64_t

In addition to atomic integer operations, the kernel also provides a family of functions that operate at the bit level.

Header File: *\<asm/bitops.h>*

These functions operate on generic pointer. There is no equivalent of the atomic integer atomic_t. A common use of the atomic integer operations is to implement counters i.e. Protecting a sole counter with a complex locking scheme is overkill, so instead developers use atomic_inc() and atomic_dec(), which are much light weight

Below functions take arguments:
	* Bit Number: 0 - 31 for 32 bit machines and 0 - 63 for 64-bit machines
	* Pointer with valid address

```
/* Atomically set the bit nr in location starting from addr */
void set_bit(int nr, volatile unsigned long *addr);

/*Atomically clear the nr-th bit starting from addr */
void clear_bit(int nr, volatile unsigned long *addr);

/*Atomically flip the value of the nr-th bit starting from addr */
void change_bit(int nr, volatile unsigned long *addr);
```

### Atomic bit operations with return value

```
/*Atomically set the bit nr in the location starting from p, and
return old value at the nrth bit */
int test_and_set_bit(unsigned int nr, volatile unsigned long *p);

/*Atomically clear the bit nr in the location starting from p, and
return old value at the nrthbit*/
int test_and_clear_bit(unsigned int nr, volatile unsigned long *p);

/*Atomically flip the bit nr in the location starting from p, and
return old value at the nrth bit*/
int test_and_change_bit(unsigned int nr, volatile unsigned long *p);

```

## Problem with Atomic instructions:
Atomic Instructions can only work with CPU word and double word size, but cannot work with shared data structures of custom size. In real life, critical regions can be more than one line. And these code paths such execute atomically to avoid race condition. To ensure atomicity of such code blocks locks are used. 
