# FreeBSD and Processor Trends

As we approach 2019 and commodity core counts continue to rise it's worth
looking in to how well FreeBSD can take advantage of larger core count and
multi-socket systems. The ThreadRipper 2 will make 32-core (64 threads) systems
commodity and EPYC2 will make it possible to have 128 cores (256 threads) in a
dual socket system. FreeBSD has always existed at the "knee" of the hardware 
commodity curve. As the definition of "commodity" moves, FreeBSD needs to 
keep pace to maintain its relevance in the server space.

Scalability can be defined on a number of axes \[Culler99\]: 
 - Problem-Constrained `PC` - The user wants to use a larger machine to solve the same problem faster.
   ```
   Speedup(n processors) = Time(1 processor) / Time(n processors)
   ```
 - Time-Constrained `TC` - the time to execute a given workload remains constant, 
   user wants to solve the larges problem possible.
   ```
   Speedup(n processors) = Work(n processors) / Work(1 processor)
   ```
 - Memory-Constrained `MC` - The user wanst to solve the largest problem that will fit in memory.
   ```
   Speedup(n processors) = Work(n processors) / Time(n processors) * Time(1 processor) / Work(1 processor =  
   Increase In Work / Increase in Execution Time
   ```
`PC` roughly translates to what is defined as `strong scaling` - as the number of
processors available to complete a task increases the extent to which the time
to complete the task decreases. `TC` is equivalent to `weak scaling` - the degree
to which the amount of work accomplished increases as the number of processors 
increases. For the purposes of this article, when we refer to scaling will be speaking of `weak scaling`. 
Scalability will be measured by the aggregate number of operations performed
during benchmarks. In general, extrapolating from these scalability measurements
to actual application performance is fraught with pitfalls as
performance bottlenecks are application and workload specific. Nonetheless,
the OS impact on any given workload can be chracterized as a combination
of the time spent in system calls and the impact of scheduling decisions. The
performance and scaling of the former can easily be characterized through
microbenchmarks. The latter can measured to a lesser degree by measuring
the impact of scheduling decisions on simple workloads.

In this article I'll talk a little bit about what makes scaling difficult
and what some of the approaches are for addressing the issues we face. I'll
then compare FreeBSD 11.1 with recent -CURRENT to show where we've made
progress over the last year and then compare the latter with performance
of the latest CentOS and Linux releases to show where the gaps are. I'll
conclude by talking a little bit about what work needs to be done where
to bridge the gaps.

From 50,000 feet there are two factors that define scaling: serialization and
scheduling. Ignoring the overhead of locking primitives themselves, serialization
overhead can roughly be defined as reducing throughput from 1 to 1/n where n 
is the number of threads waiting to acquire a lock. Scheduling overhead is more 
difficult to define. In an ideal world a given thread would only ever run on one 
core, any other threads that it communicated with would be on the same "core 
complex" - sharing an L3 cache so that IPIs (inter processor interrupts - a 
facility to allow a cpu to interrupt other cpus) and cache coherency traffic would 
not have to traverse an interconnect and any misses could be refilled without going
to memory. In practice this is impossible in the general case as CPUs are commonly
oversubscribed and the scheduler cannot infer relationships between threads in
different processes. The further away one cpu is from another the poorer the
performance for latency sensitive operations such as networking and synchronous IPC.
And the larger the distance between two cpus, the greater cost of refilling the
caches when a thread is migrated.

The scaling challenges stem from three factors:
 - memory latency
 - limits to coherency traffic
 - shared globally unique resources

To a large degree the scaling solutions are all a combination of:
 - per cpu resources
 - relaxing constraints
 - distinguishing between existence guarantees and mutual exclusion

The first two scaling challenges, memory latency and the bounds on coherency traffic,
are fundamental to the evolution of computer hardware over the last decade as the 
design artifacts formerly only seen in high end systems make their way even in to
consumer CPUs like AMD's ThreadRipper. The shared memory programming model is 
becoming an increasingly leaky abstraction. Cache coherence logic in processors
provides the single-writer /multiple-reader `SWMR` guarantees that programmers are
all accustomed to \[Sorin11\]. However, at its limit, the observed performance is defined by the
actual implementation of a distributed memory with all updates performed by message 
passing \[Hacken09\], \[Molka15\]. Performance being dictated by the message latency and bandwidth of this
machinery.





One generally progresses through distinct levels as one increases the
number of hardware threads that one is trying to exploit. Generally the
order is:
  - locking granularity
  - oversharing
  - using locks to provide existence guarantees
  - using atomic references to provide existence guarantees
  - poor cache locality between L3 caches
    or NUMA domains

Locking granularity is in essence the relative size of the scope 
that a single lock protects. The most coarse grained was a single
lock serializing all access to the kernel "Big Kernel Lock" or 
"BKL" in Linux or "Giant" in FreeBSD. This was gradually improved to
locking individual subsystems, then individual data structures, then 
fields in data structures. Even with fine grained locking, not infrequently
do we run in to the case of a widely referenced 
global resource (memory, routing table entry, etc) that can only be
accessed one at a time. Early on in SMP efforts this can easily be
addressed by moving from serializing all work in a subsystem (e.g.
networking) to locking individual sockets or in memory allocation
by periodically allocating in bulk to individual per-cpu caches where
the majority of allocations can be satisfied by lighter weight
mechanisms from per the per-cpu cache.

Oversharing is a specific manifestation of coarse locking granularity.
This is when a (typically) global lock is used to serialize access
to a resource that is global that does not need to be. An example of
this can be found in the logging of records by hwpmc (in-kernel support
for hardware performance monitoring counters) in 11 vs 12. In
11 all cpus log their respective samples to a single global buffer that
is protected by a global spin lock. On 12, each CPU has its own logging
buffer - thus requiring no locking to prevent corrupt updates, only the
disabling of interrupts to protect against preemption by another thread or
interruption by the timer interrupt which copies samples recorded by pmc's 
NMI interrupts.


One can use a lock to guarantee that
entries in a system global or process global structure has not been freed while
a thread is referencing a table entry. One example in 11.x vs 12.x is how
existence is guaranteed for connection state within the per protocol hash
table. On 11.x a thread is guaranteed that any connection found in the
table is a valid connection by requiring that all table readers do a
shared (for read) acquisition of a per-table reader/writer lock. This
allowed multiple simultaneous readers while preventing any table updates.
This was straightforward to reason about, but it is a stronger guarantee
than is required and it comes at a substantial price. In 12.x we weakened
the guarantee to only guaranteeing that any connection found during a
lookup had not been freed. Lookups are protected with epoch and updates
are serialized with a mutex. As before the lookup returns the connection
locked to guarantee existence past lookup - but now, once the lock is
acquired lookup now checks that the connection has not had the `INP_FREED`
flag set. If the flag is set it indicates that connection is pending free
and so we drop the lock and return NULL as if no connection had been found.
This change adds some additional complexity to readers, but in exchange we
no longer require a global atomic for the rwlock \[App. A\]
and updates can proceed in parallel with lookups (lookups no longer block
on updates and vice versa). This change provided a 10-20x reduction in time
spent in lookups on a loaded multi-socket server.

A more performant alternative to using locks for guaranteeing existence - but
still one that nonetheless does not scale to multiple coherency domains - is
atomically updating a reference counter for the object. Each new thread or
object holding a pointer to the object increments the reference. When the
reference is removed from the object or the thread's reference goes out of
scope the reference is decremented. When the count goes to zero the referenced
object is freed. For an object frequently referenced by many threads the
coherency traffic invalidating and migrating the cache line between LLCs
quickly becomes a bottleneck. There are 2 separate issues to unpack here: 
- Is reference counting necessary here?
- Can anything be done to make reference counting cheaper?

Interestingly enough, for stack local references, reference counting isn't actually
necessary. SMR "Safe Memory Reclamation" techniques such as Epoch Based Reclamation \[Fraser04\],
Hazard Pointers \[Michael04\] \[Hart06\] etc can allow us to provide existence guarantees without any shared
memory modifications. And reference counting can, in many cases, be made much cheaper.

Recent work in UDP to expand the scope of objects tied to the network stack's epoch
structure is now also used to guarantee existence of interface addresses. This now 
means that references to them that are stack local, where a reference was previously
acquired at entry and released before return, now no longer need to update the object's
refcount any more.

The different approaches to scalable reference counting rely on the insight that the
observed reference count can safely be different from the "true" reference count if 
we can safely handle zero detection correctly. Although there are other approaches to
this in the literature \[Ellen07\], the ones I consider most interesting are Linux's percpu 
refcount \[Corbet13\] and Refcache \[Clem13\]. The former is a per-cpu counter that degrades to a traditional
atomically updated reference count when the initial reference holder "kills" the perpcpu
refcount. This is simple and, if the life cycle of the object closely mirrors that of the
initial reference holder, can be extremely lightweight. It does not work well if the
object substantially outlives the initial owner. Refcache works by maintaining a per cpu 
cache of reference updates and then flushing them when there is conflict or at the end of 
an "epoch" where an "epoch" is several milliseconds. Zero detection is done by putting 
the object on a per-cpu "review" list when its global reference count reaches zero. The
global reference count can be assumed to be the true reference count when it has remained
at zero for two "epochs". Refcache doesn't rely on an initial reference holder with a 
closely correlated life cycle to avoid a degraded state, making it much more general in
some respects. However, the potentially multiple passes through the review queue can add
substantial overhead to the zero detection process and the latency between initial candidate
for free and final release makes it unsuitable for objects with a high rate of turnover. 
A 10 ms backlog of network mbufs or VM page structures could incur punitive overhead.

Last on this list of notable scaling anti-patterns is poor cache locality. This can be as
simple as packing structures contiguously (as opposed to a linked list) so that the prefetcher 
can furnish the next element as a thread iterates through them. However, at high operation 
rates, the way in which fields are ordered within a structure can make a decisive difference
in measured performance. A 45% increase in brk calls per second was measured when a 
reorganization of the core memory allocation structure reduced from three to two the number 
of cache lines for the most commonly accessed fields. Once serialization bottlenecks are 
eliminated, kernel performance is determined by the frequency of cache misses.

Whereas the ideals we strive for with serialization and locality are easy to define - minimal
sharing and cache misses - optimal scheduling is essential impossible to define. One can add
more knowledge about data structures and sharing to the scheduler, but that in turn slows 
down scheduling decisions. Two communicating threads with a small working set will
benefit from sharing an L2 cache, two communicating threads with a larger working set will
be adversely impacted by sharing an L2 cache. A particularly egregious example of where FreeBSD
falls down due to thread scheduling on multiple sockets is measured throughput on TCP
connections to localhost. On a ~3Ghz single socket system FreeBSD can achieve 50-60Gbps, partly
by offloading network processing to _the_ `netisr` thread. On a dual socket system of the same
clock speed, the measured throughput drops to 18-32Gbps. The worst case is when the two 
communicating processes are on one socket and the `netisr` thread is on the other so the 
notification for every packet has to cross the interconnect between sockets. At least on a dual
socket, Linux does network processing inline (i.e. no service thread) when doing TCP to localhost. 
Thus, although it achieves lower throughput than FreeBSD does on a single socket, provided both
processes are scheduled on the same socket it achieves a consistent 35Gbps. There are a number
of issues to unpack here: the existence of only one netisr thread for the entire system, where the
sender, receiver, and `netisr` thread should be scheduled, and how to convey to the scheduler that
the three different threads are communicating with each other. Nonetheless, the key take away here
should be that latency can determine usable bandwidth and poor scheduling decisions can have a
devastating impact on performance when we move from single socket to dual socket.

For users who understands in advance what workloads they will be running, the situation is manageable.
The `cpuset` command allows one to assign processor sets to processes, restricting the choices that
the scheduler has available to it.

## Measuring Scalability
As our first step on a whirlwind tour of measuring scalability we'll run the "will-it-scale" set of
benchmarks to measure the aggregate number of operations per second. First single-threaded where
we work our way up from a single process to one for every processor thread. And then multi-threaded
with a single process, working it's way from one thread to a thread for every processor thread.
Following that we'll be looking at somewhat more real world workloads whose performance is determined
in part by parallelism in the kernel: the nginx web server serving small static objects, memcached,
and PostgreSQL.

We start with 11.1 as it is the latest release that does not have any of the recent VM scalability work
to it. Thus making it a good baseline to measure any progress against.Benchmarking on `-CURRENT`, the 
development branch will show the scalability improvements over the last year or so. CentOS 7.4 is 
representative of what the typical Linux deployment sees and kind of sets a minimum threshold for where
FreeBSD needs to be. Then last we look at Linux 4.18 - the latest Linux release to provide ceteris paribus
measurements against the Linux equivalent of the FreeBSD development branch.


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
