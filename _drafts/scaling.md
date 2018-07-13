FreeBSD and Processor Trends

As we approach 2019 and commodity core counts continue to rise it's worth
looking in to how well FreeBSD can take advantage of larger core count and
multi-socket systems. The ThreadRipper 2 will make 32-core (64 threads) systems
commodity and EPYC2 will make it possible to have 128 cores (256 threads) in a
dual socket system. Measuring scalability is fraught with pitfalls as
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


Scaling challenges:
 - memory latency
 - limits to coherency traffic
 - shared globally unique resources

Scaling solutions:
 - per cpu resources
 - relaxing constraints
 - distinguishing between liveness and mutual exclusion


One generally progresses through distinct levels as one increases the
number of hardware threads that one is trying to exploit. Generally the
order is:
  - locking granularity
  - oversharing
  - using locks to guarantee liveness
  - using atomics to guarantee liveness
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
this can be found in the logging of records by hwpmc in 11 vs 12. In
11 all cpus log their respective samples to a single global buffer that
is protected by a global spin lock. On 12, each CPU has its own logging
buffer - thus requiring no locking to prevent corrupt updates, only the
disabling of interrupts to protect against preemption by another thread or
interruption by the timer interrupt which copies samples recorded by pmc's 
NMI interrupts.

An object is said to be "live" if the object's fields are valid and the 
object itself has not been freed. One can use a lock to guarantee that
entries in a system global or process global structure has not been while
a thread is referencing a table entry. One example in 11.x vs 12.x is how
liveness is guaranteed for connection state within the per protocol hash
table. On 11.x a thread is guaranteed that any connection found in the
table is a valid connection by requiring that all table readers do a
shared (for read) acquisition of a per-table reader/writer lock. This
allowed multiple simultaneous readers while preventing any table updates.
This was straightforward to reason about, but it is a stronger guarantee
than is required and it comes at a substantial price. In 12.x we weakened
the guarantee to only guaranteeing that any connection found during a
lookup had not been freed. Lookups are protected with epoch and updates
are serialized with a mutex. As before the lookup returns the connection
locked to guarantee liveness past lookup - but now, once the lock is
acquired lookup now checks that the connection has not had the `INP_FREED`
flag set. If the flag is set it indicates that connection is pending free
and so we drop the lock and return NULL as if no connection had been found.
This change adds some additional complexity to readers, but in exchange we
no longer require a global atomic for the rwlock [explain this to the reader]
and updates can proceed in parallel with lookups (lookups no longer block
on updates and vice versa). This change provided a 10-20x reduction in time
spent in lookups on a loaded multi-socket server.

A more performant alternative to using locks for guaranteeing liveness - but
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
necessary. SMR "Safe Memory Reclamation" techniques such as Epoch Based Reclamation,
Hazard Pointers etc can allow us to provide liveness guarantees without any shared
memory modifications. And reference counting can, in many cases, be made much cheaper.

Recent work in UDP to expand the scope of objects tied to the network stack's epoch 
structure is now also used to guarantee liveness of interface addresses. This now 
means that references to them that are stack local, where a reference was previously
acquired at entry and released before return, now no longer need to update the object's
refcount any more.

The different approaches to scalable reference counting rely on the insight that the
observed reference count can safely be different from the "true" reference count if 
we can safely handle zero detection correctly. Although there are many approaches to
this in the literature [...], the ones I consider most interesting are Linux's percpu 
refcount and Refcache. The former is a per-cpu counter that degrades to a traditional
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
of issues to unpack with this issue....



## Appendix A - FreeBSD Serialization Primitives

#### mutex

Mutex is the most straightforward primitive. There are two classes, `MTX_DEF` and `MTX_SPIN`. A `MTX_SPIN` 
mutex is considered "heavyweight" because it disables interrupts. It can be acquired in any context. While
it is held (and with interrupts disabled in general) a thread can only acquire other `MTX_SPIN` locks and 
cannot allocate memory or sleep. While holding a `MTX_DEF` lock a thread can do anything that would not 
entail sleeping (acquire either types of mutex, rwlocks, non-sleepable memory allocations etc.). While 
waiting to acquire a `MTX_SPIN` mutex a thread will "spin" polling the lock for release by its current holder.
While waiting to acquire a `MTX_DEF` a thread will "adaptively spin" on it, polling for release if the 
current holder is running and being enqueued a `turnstile` if the current holder has been preempted. 
`Turnstiles` are faciility for priority propagation allowing blocked threads to "lend" their scheduler
priority to the current lock holder as a mechanism for avoiding priority inversion.


#### rwlock



#### sx

#### rmlock

#### lockmgr

#### epoch





