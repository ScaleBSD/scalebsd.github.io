There is a suite of microbenchmarks called will-it-scale that runs N parallel instances of a thread or process
all performing the same test case. It measures the number of testcase loops in aggregate and the minimum and
maximum number achieved by a given thread or process. This is useful for zooming in on individual system
bottlenecks.

Today I'm looking at mmap1_processes. Literally all it does is repeatedly mmap and then munmap a range of
memory.

```
void testcase(unsigned long long *iterations, unsigned long nr)
{
        while (1) {
                char *c = mmap(NULL, MEMSIZE, PROT_READ|PROT_WRITE,
                               MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
                assert(c != MAP_FAILED);
                munmap(c, MEMSIZE);

                (*iterations)++;
        }
}
```

I have 96 cpus and I want to get 5 samples:

```
mmacy@anarchy [~/devel/will-it-scale|22:11|4] ./mmap1_processes -t 96 -s 5
testcase:Anonymous memory mmap/munmap of 128MB
warmup
min:9032 max:16299 total:1203113
min:7529 max:14293 total:1007495
min:7856 max:13651 total:1013123
min:7559 max:14663 total:1015132
min:6837 max:13661 total:1012563
min:7785 max:13609 total:1011908
measurement
min:7864 max:13726 total:1005918
min:7876 max:14460 total:1006319
min:7722 max:13627 total:1009119
min:7378 max:14145 total:1013143
min:7615 max:12599 total:1009155
average:1008730
```

Somewhat interestingly performance actually peaks at 12 processes, plateaus until 24, and then rapidly declines.
```
mmacy@anarchy [~/devel/will-it-scale|22:15|10] ./mmap1_processes -t 12 -s 5
...
min:120480 max:165747 total:1722483
average:1719430
```

A simple sample of `UNHALTED_CORE_CYCLES` will tell us where the system is spending its time.
![](/media/svg/2018.05.02/mmap1_master.svg)

It appears that the we're actually rate limited by munmap as opposed to mmap and all of our time is being spent in
`pmap_remove()`. `pmap_remove()` calls `pmap_delayed_invl_started()`.

```
/*
 * Start a new Delayed Invalidation (DI) block of code, executed by
 * the current thread.  Within a DI block, the current thread may
 * destroy both the page table and PV list entries for a mapping and
 * then release the corresponding PV list lock before ensuring that
 * the mapping is flushed from the TLBs of any processors with the
 * pmap active.
 */
static void
pmap_delayed_invl_started(void)
{
	struct pmap_invl_gen *invl_gen;
	u_long currgen;

	invl_gen = &curthread->td_md.md_invl_gen;
	PMAP_ASSERT_NOT_IN_DI();
	mtx_lock(&invl_gen_mtx);
	if (LIST_EMPTY(&pmap_invl_gen_tracker))
		currgen = pmap_invl_gen;
	else
		currgen = LIST_FIRST(&pmap_invl_gen_tracker)->gen;
	invl_gen->gen = currgen + 1;
	LIST_INSERT_HEAD(&pmap_invl_gen_tracker, invl_gen, link);
	mtx_unlock(&invl_gen_mtx);
}
```


So what we see is that every call to `pmap_remove()` is globally serialized on the `invl_gen_mtx` mutex. Processes
without shared mappings running even on different sockets all have to acquire the same lock to track the pmap
invalidation generation. The point of this is to ensure that we block processes from accessing mappings that may
be about to be invalidated.

```
/*
 * Ensure that all currently executing DI blocks, that need to flush
 * TLB for the given page m, actually flushed the TLB at the time the
 * function returned.  If the page m has an empty PV list and we call
 * pmap_delayed_invl_wait(), upon its return we know that no CPU has a
 * valid mapping for the page m in either its page table or TLB.
 *
 * This function works by blocking until the global DI generation
 * number catches up with the generation number associated with the
 * given page m and its PV list.  Since this function's callers
 * typically own an object lock and sometimes own a page lock, it
 * cannot sleep.  Instead, it blocks on a turnstile to relinquish the
 * processor.
 */
static void
pmap_delayed_invl_wait(vm_page_t m)
{
	struct turnstile *ts;
	u_long *m_gen;

	m_gen = pmap_delayed_invl_genp(m);
	while (*m_gen > pmap_invl_gen) {
		ts = turnstile_trywait(&invl_gen_ts);
		if (*m_gen > pmap_invl_gen)
			turnstile_wait(ts, NULL, TS_SHARED_QUEUE);
		else
			turnstile_cancel(ts);
	}
}
```

Before we jump to conclusions we dig in to when and why the change was made.
```
commit b4ddfd616790955ff2b6f179064ca000419a56c2
Author: kib <kib@FreeBSD.org>
Date:   Sat May 14 23:35:11 2016 +0000

    Eliminate pvh_global_lock from the amd64 pmap.
    
    The only current purpose of the pvh lock was explained there
    On Wed, Jan 09, 2013 at 11:46:13PM -0600, Alan Cox wrote:
    > Let me lay out one example for you in detail.  Suppose that we have
    > three processors and two of these processors are actively using the same
    > pmap.  Now, one of the two processors sharing the pmap performs a
    > pmap_remove().  Suppose that one of the removed mappings is to a
    > physical page P.  Moreover, suppose that the other processor sharing
    > that pmap has this mapping cached with write access in its TLB.  Here's
    > where the trouble might begin.  As you might expect, the processor
    > performing the pmap_remove() will acquire the fine-grained lock on the
    > PV list for page P before destroying the mapping to page P.  Moreover,
    > this processor will ensure that the vm_page's dirty field is updated
    > before releasing that PV list lock.  However, the TLB shootdown for this
    > mapping may not be initiated until after the PV list lock is released.
    > The processor performing the pmap_remove() is not problematic, because
    > the code being executed by that processor won't presume that the mapping
    > is destroyed until the TLB shootdown has completed and pmap_remove() has
    > returned.  However, the other processor sharing the pmap could be
    > problematic.  Specifically, suppose that the third processor is
    > executing the page daemon and concurrently trying to reclaim page P.
    > This processor performs a pmap_remove_all() on page P in preparation for
    > reclaiming the page.  At this instant, the PV list for page P may
    > already be empty but our second processor still has a stale TLB entry
    > mapping page P.  So, changes might still occur to the page after the
    > page daemon believes that all mappings have been destroyed.  (If the PV
    > entry had still existed, then the pmap lock would have ensured that the
    > TLB shootdown completed before the pmap_remove_all() finished.)  Note,
    > however, the page daemon will know that the page is dirty.  It can't
    > possibly mistake a dirty page for a clean one.  However, without the
    > current pvh global locking, I don't think anything is stopping the page
    > daemon from starting the laundering process before the TLB shootdown has
    > completed.
    >
    > I believe that a similar example could be constructed with a clean page
    > P' and a stale read-only TLB entry.  In this case, the page P' could be
    > "cached" in the cache/free queues and recycled before the stale TLB
    > entry is flushed.
    
    TLBs for addresses with updated PTEs are always flushed before pmap
    lock is unlocked.  On the other hand, amd64 pmap code does not always
    flushes TLBs before PV list locks are unlocked, if previously PTEs
    were cleared and PV entries removed.
    
    To handle the situations where a thread might notice empty PV list but
    third thread still having access to the page due to TLB invalidation
    not finished yet, introduce delayed invalidation.  Comparing with the
    pvh_global_lock, DI does not block entered thread when
    pmap_remove_all() or pmap_remove_write() (callers of
    pmap_delayed_invl_wait()) are executed in parallel.  But _invl_wait()
    callers are blocked until all previously noted DI blocks are leaved,
    thus ensuring that neccessary TLB invalidations were performed before
    returning from pmap_remove_all() or pmap_remove_write().
    
    See comments for detailed description of the mechanism, and also for
    the explanations why several pmap methods, most important
    pmap_enter(), do not need DI protection.
```

TL;DR a much larger section of code was previously globally serialized on the
pvh lock and this narrows it down to a couple of code paths. However, what we're trying to achieve ...
not allowing callers of pmap_delayed_invl_wait to continue until the invalidate global invalidate
generation catches up with that of the page passed - thus ensuring any caller invalidating it has
left its critical section. This is essentially what EBR (Epoch Based Reclamation) does: 
* Upon entry into a read-side protected section, readers set an 
     active bit and take a snapshot of a global epoch counter. 

 * Synchronize operations increment the epoch counter only if 
     all active threads have a snapshot of the latest value of the 
     global counter. 

 * If the epoch counter is successfully incremented twice from 
     the time synchronize was called, then no references could 
     exist to objects logically deleted before the synchronize call.

The general
idea is that we don't want to free a resource (or in this case reference it) until all threads are
done referencing it. There are existing primitives in the kernel from [ConcurrencyKit](http://concurrencykit.org/) that implement
this in a more general fashion that have been widely used at scale. See [EBR](http://concurrencykit.org/presentations/ebr.pdf) for more details.



As an experiment, I implemented a basic wrapper for the ck primitives, the core of which are `epoch_enter()`, `epoch_exit()`, and `epoch_wait()` and then just changed the `pmap_delayed_invl_started()`,
`pmap_delayed_invl_finished()`, `pmap_delayed_invl_wait()` to call `epoch_enter()`, `epoch_exit()`, and `epoch_wait()`.

```
mmacy@anarchy [~/devel/will-it-scale|22:56|2] ./mmap1_processes -t 96 -s 5
...
min:34124 max:39816 total:3548585
average:3550909
```

```
x before
+ after
    N           Min           Max        Median           Avg        Stddev
x   5       1005918       1013143       1009119     1008730.8     2895.1156
+   5       3548585       3552587       3552085       3550909     2039.8783
Difference at 95.0% confidence
        2.54218e+06 +/- 3652.34
        252.018% +/- 1.06257%
        (Student's t, pooled s = 2504.28)
```

Looking at `UNHALTED_CORE_CYCLES` sample now we see that swap reservation accounting is now the bottleneck.
[![](/media/svg/2018.05.02/mmap1_vmhack.svg)](/media/svg/2018.05.02/mmap1_vmhack.svg)

This too is fixable, but a topic for another day.

