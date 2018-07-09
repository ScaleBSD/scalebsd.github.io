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


Scaling challenges:
 - memory latency
 - limits to coherency traffic
 - shared globally unique resources

Scaling solutions:
 - per cpu resources
 - relaxing constraints
 - distinguishing between liveness and mutual exclusion


One generally progressives through distinct levels as one increases the
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




