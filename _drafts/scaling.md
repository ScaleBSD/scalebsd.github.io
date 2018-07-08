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

Locking granularity is in essence having one lock being simultaneously
accessed by many different hardware threads. This stems from having a
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
disabling of interrupts to protect against interruption by the timer
interrupt which copies samples recorded by NMI interrupts.

Using locke


