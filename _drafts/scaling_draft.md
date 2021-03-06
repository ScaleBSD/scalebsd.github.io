# FreeBSD and Processor Trends
 
	At Computex 2018, Intel unveiled a prototype 28-core system. Within a few months, AMD launched the world’s most powerful desktop processor, the ThreadRipper 2 featuring 32-core(64 thread). According to AMD, “2nd Gen AMD Ryzen Threadripper 2990WX delivers up to 53% faster multi-thread performance than the Core i9-7980XE”. AMD’s EPYC2 is in the lab and rumored to enable 128 core (256 thread) in a dual socket system. Historically, FreeBSD has existed at the "knee" of the hardware commodity curve. In order to maintain its relevance in the server space, FreeBSD needs to keep pace with the latest processor innovations.

Scalability 
	As a property of systems, is generally difficult to define[2] and in any particular case it is necessary to define the specific requirements for scalability on those dimensions that are deemed important. Scalability can be measured by the issues that constrain it, such as 

 - Problem-Constrained Scalability, also referred to as “strong scaling” - The ability to solve a problem faster by using more processors. As the number of processors available to complete a task increases, the time to complete the problem decreases:
   ```
   Speedup(n processors) = Time(1 processor) / Time(n processors)
   ```
- Memory-Constrained Scalability – The ability to solve the largest problem size that will fit in memory. 
    ```
   Speedup(n processors) = Work(n processors) / Time(n processors) * Time(1 processor) / Work(1 processor =  
   Increase In Work / Increase in Execution Time
   ```
 - Time-Constrained Scalability, also referred to as “weak scaling” - The ability to solve the largest problem size within a fixed time. It is the degree to which the amount of work accomplished increases as the number of processors increases:
   
   ```
   Speedup(n processors) = Work(n processors) / Work(1 processor)
   ```
   ```
In this article, scalabilty refers to time-constrained scalabilty (“weak scaling”) and is characterized by the aggregate number of operations performed during benchmarks. Performance bottlenecks are application and workload specific. Therefore it is problematic to extrapolate actual application performance from these scalability measurements. Nonetheless, the OS impact on any given workload can be characterized as a combination of the impact of scheduling decisions and average time per system call. System benchmarking can measure the performance and scalabilty of scheduling decisions by measuring their impact on simple workloads. 
Microbenchmarks seek to measure the performance of small and isolated operations. 
As such, microbenchmarks are helpful to characterize the time spent in system calls. It is important for the reader to understand that the purpose
of microbenchmarks in this particular case is not to measure workloads themselves. They seek to observe the scaling of individual OS services to measure scalabilty in isolation. These measurements are only predictive of performance on real world workloads to the extent to which a workload uses the individual service being measured.




## What Makes Scaling Difficult –- Serialization and Scheduling
Serialization overhead can roughly be defined as reducing throughput from 1 to 1/n. In this case, n is the number of threads waiting to acquire a lock and the minimal overhead of locking primatives is not included. Scheduling overhead is more 
difficult to define. In an ideal world a given thread would run exclusively on a single core and any other communicating threads would run on the same "core 
complex" - sharing an L3 cache. Utilizing this structure, the IPIs (inter processor interrupts - a facility to allow a cpu to interrupt other cpus) and cache coherency traffic would not traverse an interconnect. Any misses could be refilled without  fetching data from memory. Unfortunately, in practice this scenario is impossible. Generally, CPUs are oversubscribed and the scheduler cannot infer relationships between threads in different processes. Increased physical distance between cpus increases the cost of refilling the caches upon thread migration. It also results in poorer performance for latency sensitive operations such as networking and synchronous IPC.

Scaling challenges arise from three factors:
 - Memory latency
 - Limits to coherency traffic
 - Shared globally unique resources

Scaling solutions are mostly a combination of:
 - Per cpu resources
 - Relaxing constraints
 - Distinguishing between existence guarantees and mutual exclusion

Memory latency and the bounds on coherency traffic, are fundamental to the evolution of computer hardware over the last decade. What were once design artifacts appearing exclusively in high end systems have become important considerations even in consumer CPUs like AMD's ThreadRipper. The shared memory programming model is becoming an increasingly leaky abstraction. Cache coherence logic in processors provides the single-writer /multiple-reader `SWMR` guarantees that programmers are all accustomed to \[Sorin11\]. However, at its limit, the observed performance is defined by the actual implementation of a distributed memory with all updates performed by message passing \[Hacken09\], \[Molka15\]. Today, message latency and bandwidth are dominant factors in observed performance.

Consequences of increasing the number of hardware threads:
- Locking granularity
- Oversharing
- Using locks to provide existence guarantees
- Using atomic references to provide existence guarantees 
- Poor cache locality between L3 caches or NUMA domains.

### Locking Granularity – 
Locking granularity refers to how many operations are protected by a single lock.
The “Big Kernel Lock” or “BKL” in Linux or “Giant” in FreeBSD initially encompassed the entire kernel in a single lock. This evolved into locks for individual subsystems, then individual data structures, and finally fields in data structures. Even with fine grained locking, the case of a widely referenced global resource (memory, routing table entry, etc) that can only be accessed one at a time occurs. In general, locking granularity in FreeBSD is already relatively fine grained.

### Oversharing - 
Oversharing refers to a specific manifestation of coarse locking granularity.
This typically occurs when a global lock is used to serialize access to a global resource that does not need a global scope. An example is the logging of records by hwpmc (in-kernel support for hardware performance monitoring counters) in 11 vs 12. In 11 all cpus log their respective samples to a single global buffer that is protected by a global spin lock. While in 12, each CPU has its own logging buffer. The only serialization required is disabling interrupts to prevent preemption by another thread or the timer interrupt responsible for copying samples. During  sampling, this change completely eliminated contention related overhead.
 
### Locking for existence guarantees
Locks can guarantee that an entry in either a system global or process global structure has not been freed while a thread is referencing a table entry. One example in FreeBSD 11.x vs FreeBSD 12.x is how existence is guaranteed for connection state within the per protocol hash table. FreeBSD 11.x guarantees a thread that any connection found in the table is valid by requiring that all table readers do a shared (for read) acquisition of a per-table reader/writer lock. This allowed multiple simultaneous readers while preventing any table updates.
Although conceptually strightforward, this comes at a substantial price and the guarantee is stronger than required. FreeBSD 12.x weakens the guarantee to provide that any connection found during a lookup had not been freed. Lookups are protected with epoch and updates are serialized with a mutex. The policy that the 
lookup returns the connection locked to guarantee existence past lookup remains unchanged. However, once the lock is acquired lookup now checks that the connection has not had the `INP_FREED` flag set. If the flag is set, this indicated that connection is pending free. In this case, we drop the lock and return NULL as if no  connection had been found. This change adds some additional complexity to readers, but in exchange we no longer require a global atomic for the rwlock \[App. A\]
and updates can proceed in parallel with lookups (lookups no longer block on updates and vice versa). This change provided a 10-20x reduction in time spent in lookups on a loaded multi-socket server.

### Atomic refcounts for existence guarantees
A better performance alternative to using locks for guaranteeing existence is atomically updating a reference counter for the object. Nonetheless it still does not scale to multiple coherency domains. Each new thread or object holding a pointer to the object increments the reference. When the reference is removed from the object or the thread's reference goes out of scope the reference is decremented. When the count goes to zero the referenced object is freed. For an object frequently referenced by many threads the coherency traffic invalidating and migrating the cache line between LLCs quickly becomes a bottleneck. There are 2 separate issues to unpack here: 
- Is reference counting necessary here?
- Can anything be done to make reference counting cheaper?

Interestingly for stack local references, reference counting isn't actually
necessary. SMR "Safe Memory Reclamation" techniques such as Epoch Based Reclamation \[Fraser04\], Hazard Pointers \[Michael04\] \[Hart06\] etc can allow us to provide existence guarantees without any shared memory modifications. In many cases, reference counting can be made much cheaper.

Recent work in UDP to expand the scope of objects tied to the network stack's epoch
structure is used to guarantee the existence of interface addresses. This now 
means that references to them that are stack local. Previously a reference was 
acquired at entry and released before return. Now there is no longer a need to update the object's refcount.

The observed reference count can safely be different from the "true" reference count if we can safely handle zero detection correctly. The different approaches to scalable reference county rely on this insight. Although there are other approaches to this in the literature \[Ellen07\], the ones I consider most interesting are Linux's percpu refcount \[Corbet13\] and Refcache \[Clem13\]. The former is a per-cpu counter that degrades to a traditional atomically updated reference count when the initial reference holder "kills" the perpcpu refcount. It’s advantage is that is simple and can be extremely lightweight provided that the life cycle of the object closely mirrors that of the initial reference holder. It does not work well if the object substantially outlives the initial owner. Refcache maintains a per cpu cache of reference updates and flushes them when there is conflict or at the end of an "epoch". An "epoch" is several milliseconds. Zero detection is done by putting the object on a per-cpu "review" list when its global reference count reaches zero. The global reference count can be assumed to be the true reference count when it has remained at zero for two "epochs". Refcache doesn't rely on an initial reference holder with a closely correlated life cycle to avoid a degraded state. In some respects this makes it much more general. However, potential multiple passes through the review queue can add substantial overhead to the zero detection process. The latency between initial candidate for free and final release makes it unsuitable for objects with a high rate of turnover. For example, 10 ms backlog of network mbufs or VM page structures could incur punitive overhead.

### Cache Locality
Poor cache locality is last on this list of notable scaling anti-patterns. A simple example of which is packing structures contiguously as opposed to a linked list so that the prefetcher can furnish the next element as a thread iterates through them.  
At high operation rates, the way in which fields are ordered within a structure can make a measurable performance difference. A 45% increase in brk calls per second was measured when a reorganization of the core memory allocation structure reduced from three to two the number of cache lines for the most commonly accessed fields. Once serialization bottlenecks are eliminated, kernel performance is determined by the frequency of cache misses.

Minimal sharing and cache misses are easily definable ideals for serialization and locality. In the case of optimal scheduling ideals are essentially impossible to define. Additional data structure knowledge and sharing afforded to the scheduler slows down scheduling decisions. When sharing an L2 cache, two communicating threads with a small working set will benefit while two communicating threads with a larger working set will be adversely impacted. A particularly egregious example of where FreeBSD falls down due to thread scheduling on multiple sockets is measured throughput on TCP connections to localhost. On a ~3Ghz single socket system FreeBSD can achieve 50-60Gbps, partly by offloading network processing to _the_ `netisr` thread. On a dual socket system of the same clock speed, the measured throughput drops to 18-32Gbps. In the worst case, two communicating processes are on one socket and the `netisr` thread is on the other. Therefore the 
notification for every packet has to cross the interconnect between sockets. At least on a dual socket, Linux does network processing inline (i.e. no service thread) when doing TCP to localhost. Linux achieves lower throughput than FreeBSD does on a single socket. However, it achieves a consistent 35Gbps provided both processes are scheduled on the same socket. There are a number of issues to unpack here: 
- The existence of only one netisr thread for the entire system
- Where the sender, receiver, and `netisr` thread should be scheduled
- How to convey to the scheduler that the three different threads are communicating with each other. 
Nonetheless, the key take away here should be that latency can determine usable bandwidth and poor scheduling decisions can have a devastating impact on performance when we move from single socket to dual socket.

For users who understands in advance what workloads they will be running, the situation is manageable. The `cpuset` command allows one to assign processor sets to processes, restricting the choices that the scheduler has available to it.




## Measuring Scalability
As our first step on a whirlwind tour of measuring scalability we'll run the "will-it-scale" set of benchmarks to measure the aggregate number of operations per second. Single-threaded and multi-threaded tests will begin with a single process and run until there is one for every processor thread. Additional tests will analyze real world workloads whose performance is determined in part by parallelism in the kernel. For example, the nginx web server serving small static objects, memcached, and PostgreSQL.

<!-- I'll then compare FreeBSD 11.1 with recent -CURRENT to show where we've made
progress over the last year and then compare the latter with performance
of the latest CentOS and Linux releases to show where the gaps are. I'll
conclude by talking a little bit about what work needs to be done where
to bridge the gaps. -->


We start with 11.1, the latest release that does not have any of the recent VM scalability work. Thus making it a good baseline to measure any progress against. Benchmarking on `-CURRENT`, the development branch will show the scalability improvements over the last year or so. CentOS 7.4 is representative of what the typical Linux deployment uses and sets an approximate minimum threshold for 
FreeBSD. Lastly Linux 4.18 - the latest Linux release to provide ceteris paribus will be analyzed. The measurements will be taken against the Linux equivalent of the FreeBSD development branch.

## Appendix A - FreeBSD Serialization Primitives

#### mutex

Mutex is the most straightforward primitive, ownership is acquired by a thread doing a compare and swap 
operation on it with a pointer to the acquiring thread's structure. If the lock already contains a thread 
pointer it is owned, if not it is free. There are two classes, `MTX_DEF` and `MTX_SPIN`. A `MTX_SPIN` 
mutex is considered "heavyweight" because it disables interrupts. It can be acquired in any context. While
it is held (and with interrupts disabled in general) a thread can only acquire other `MTX_SPIN` locks and 
cannot allocate memory or sleep. While holding a `MTX_DEF` lock a thread can do anything that would not 
entail sleeping (acquire either types of mutex, rwlocks, non-sleepable memory allocations etc.). While 
waiting to acquire a `MTX_SPIN` mutex a thread will "spin" polling the lock for release by its current holder.
While waiting to acquire a `MTX_DEF` a thread will "adaptively spin" on it, polling for release if the 
current holder is running and being enqueued a `turnstile` if the current holder has been preempted. 
`Turnstile`s are faciility for priority propagation allowing blocked threads to "lend" their scheduler
priority to the current lock holder as a mechanism for avoiding priority inversion. In other words, if 
the blocked thread is higher priority the lockholder's scheduler priority will be elevated to that of the 
blocked thread.

#### rwlock

The rwlock extends the semantics of the `MTX_DEF` mutex by supporting two modes - single writer and multiple
readers. Its implementation is similar to that of mutex with some additional state assigned to the lower bits
of the lock field. In single writer mode it behaves the same as a mutex would. In reader mode, multiple readers 
can  acquire the lock and writers are blocked until all readers drop the lock. In this mode we can no longer
efficiently track the lockholders' state so we cannot propagate priority and it is not possible for an acquirer
to know if all holders are running. Thus a thread can only spin speculatively.

It supports an arbitrary number of readers, so it's the first primitive developers have traditionally
reached for when trying to guarantee existence of fields during a table lookup. However, every read acquisition
and release involves an atomic update of the lock. When the lock is shared across core complexes (and thus
updates entail cache coherency traffic between LLCs to transition the previous holder's cacheline from 
modified/exclusive to invalid) its use can quickly become very expensive.

#### sx

The sx lock is logically equivalent to the rwlock with the critical difference being that a lockholder can sleep.
Blocked readers and writers are maintained on sleepqueues and priority propagation is not done.

#### rmlock

Like the rwlock and sx, the rmlock "read mostly lock" is a reader/writer lock. Its critical difference is that
acquisition for read is extremely fast and does not involve any atomics. Acquisition for write is extremely expensive.
In its current incarnation it involves a system wide IPI to all other cpus. This is actually a reasonable primitive
for guaranteeing existence if updates are infrequent enough. It is easy to reason about, having the same semantics
as familiar rwlocks.

#### lockmgr

Lockmgr is something of an anachronism. It has some unique features required by the VFS (virtual file system)
layer. It is generally not a bottleneck in today's code and its idiosyncracies are outside the scope of what
I hope to touch on.

#### epoch

The epoch primitive allows the kernel to guarantee that structures protected by it will remain live while a thread 
is in an epoch section. Executing `do_stuff()` in an epoch section looks something like:
```
 epoch_enter(global_epoch);
 do_stuff(); ...
 epoch_exit(global_epoch);
``` 
A thread deleting an object referenced within an epoch section can either synchronously wait for all threads in an
epoch section during the current epoch plus a grace period by calling `epoch_wait(epoch)`. Or it can enqueue the 
object to be freed at a later time using `epoch_call(epoch, context, callback)`.
allowing a service thread to confirm - at lower cost than a synchronous operation - that a grace period has elapsed.
In many respects the read side of epoch has similar characteristics to the read side of an rmlock. However, it does 
not provide a mutual exclusion guarantee. Modifications to an epoch protected data structure can proceed in parallel
with readers. Modifications do typically need to be explicitly serialized with respect to each other. Thus a mutex 
is used to protect a writer against other writers. Although its implementation and the performance tradeoffs are
completely different from Linux's RCU, it largely supports the same programming design patterns.There are two variants
of epoch, preemptible and non-preemptible. A non-preemptible epoch is lighter weight but does not permit the calling
thread to acquire any lock type other than `MTX_SPIN` mutexes. Epoch is new in 
FreeBSD 12. It is essentially in-kernel scaffolding built around ConcurrencyKit's epoch (Epoch Based Reclamation) API.


## Bibliography

\[Atti11\] Attiya, H., Guerraoui, R., Hendler, D., Kuznetsov, P., Michael, M. M., Vechev, M. 2011. Laws of 
order: expensive synchronization in concurrent algorithms cannot be eliminated. In Proceedings 
of the 38th Annual ACM SIGPLAN-SIGACT Symposium on Principles of Programming Languages: 487-498; 
http://doi.acm.org/10.1145/1926385.1926442

\[Boyd10\] S. Boyd-Wickizer, A. T. Clements, Y. Mao, A. Pesterev, M. F. Kaashoek, R. Morris, and N. Zeldovich. An analysis of Linux scalability to many cores. In Proceedings of the 9th Symposium on Operating Systems Design and Implementation (OSDI), Vancouver, Canada, Oct. 2010.

\[Corbet10\] Corbet, J. The search for fast, scalable counters, May 2010. http://lwn.net/Articles/170003/.

\[Corbet13\] Corbet, J.Per-CPU reference counts, July 2013. https://lwn.net/Articles/557478/

\[Clem13\] Clements,  A. T., M.  F.  Kaashoek,  and  N.  Zeldovich. RadixVM: Scalable address spaces for multithreaded applications. In
Proceedings of the ACM EuroSys Conference, Prague, Czech Republic, April 2013.

\[Clem14\] Clements, A. T. The scalable commutativity rule: Designing scalable software for multicore processors, Ph.D. dissertation, Massachusetts Institute of Technology, Jun. 2014. [Online]. Available: https://pdos.csail.mit.edu/papers/aclements-phd.pdf

\[Culler99\] Culler, D. Singh, J. P., Gupta, A. Parallel Computer Architecture - A Hardware / Software Approach,
Morgan Kaufman, 1999

\[Ellen07\] F. Ellen, Y. Lev, V. Luchango, and M. Moir. SNZI: Scalable nonzero indicators. In Proceedings of the 26th ACM SIGACT-SIGOPS Symposium on Principles of Distributed Computing, Portland, OR, Aug. 2007.

\[Fraser04\] Fraser, K. Practical lock-freedom, Ph.D. Thesis, University of Cambridge Computer Laboratory, 2004

\[Hacken09\] D. Hackenberg,  D. Molka,  and W. Nagel.   Comparing cache architectures and coherency protocols on x86-64 multicore SMP systems.  MICRO 2009, pages 413–422.

\[Hart07\] Hart, T. E., McKenney, P. E., Demke Brown, A., Walpole, J. 2007. Performance of memory reclamation for lockless synchronization. Journal of Parallel and Distributed Computing 67(12): 1270-1285; 
http://dx.doi.org/10.1016/j.jpdc.2007.04.010

\[Herl08\] Herlihy, M., Shavit, N. 2008. The Art of Multiprocessor Programming. San Francisco: Morgan Kaufmann Publishers Inc. 

\[McKen11\] McKenney, P. E. 2011. Is parallel programming hard, and, if so, what can you do about it? kernel.org; 
https://www.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html 
 
\[Michael04\] Michael, M. M. Hazard pointers: safe memory reclamation for lock-free
objects, IEEE Trans. Parallel Distrib. Syst. 15 (6) (2004) 491–504.

\[Molka15\] Daniel Molka, Daniel Hackenberg, Robert Schöne, and Wolfgang E Nagel. 2015.
Cache Coherence Protocol and Memory Performance of the Intel Haswell-EP
Architecture. In Parallel Processing (ICPP), 2015 44th International Conference on.IEEE, 739–748.

\[Sorin11\] D. J. Sorin, M. D. Hill, and D. A. Wood. A Primer on Memory Consistency and Cache Coherence. Morgan and Claypool, 2011.
  
\[Wang16\] Q. Wang, T. Stamler, and G. Parmer, “Parallel sections: Scaling system-level data-structures,” in
Proceedings of the ACM EuroSys Conference, 2016
