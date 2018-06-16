---
layout: post
title: "UDP and providing liveness guarantees with epoch"
date: 2018-06-16
---
Up until very recently UDP had some serious scaling issues on FreeBSD. The
changes taken to address this are an interesting demonstration of using
epoch over rwlocks and atomics for guaranteeing liveness.


Running 64 netperf clients `netperf -H $DUT -t UDP_STREAM -- -m 1` achieves 590kpps and yields an activity profile like this:
[![](/media/svg/2018.05.11/udpsender.svg)](/media/svg/2018.05.11/udpsender.svg)


Actually setting the flowid on the inpcb:
```
--- a/sys/netinet/udp_usrreq.c
+++ b/sys/netinet/udp_usrreq.c
@@ -1592,6 +1592,8 @@ udp_attach(struct socket *so, int proto, struct thread *td)
        inp = sotoinpcb(so);
        inp->inp_vflag |= INP_IPV4;
        inp->inp_ip_ttl = V_ip_defttl;
+       inp->inp_flowid = arc4random();
+       inp->inp_flowtype = M_HASHTYPE_OPAQUE;
 
        error = udp_newudpcb(inp);
        if (error) {
```

Yields 1.1Mpps and looks like:
[![](/media/svg/2018.05.11/udpsender2.svg)](/media/svg/2018.05.11/udpsender2.svg)

Changing the global rwlock read lock path for the global interface list to be epoch protected yields 1.8-2.5Mpps and looks this:
[![](/media/svg/2018.05.11/udpsender3.svg)](/media/svg/2018.05.11/udpsender3.svg)

We're now contending heavily on updating the refcount for the output interface address.We can fix this by relying on epoch to guarantee liveness for short lived references.

Although we've now increased our singled threaded throughput to 1.18Mpps, 64 netperf jobs actually have lower overall throughput (1.9 Mpps) because we're now contending on fewer locks:
[![](/media/svg/2018.05.11/udpsender4.svg)](/media/svg/2018.05.11/udpsender4.svg)

If we replace the if_afdata rlock path protecting the liveness of L2 table entries (ARP or ND6) we get back up to 2.4-2.6Mpps,
but fully half the samples are now attributable to the inpcbhash read lock:
[![](/media/svg/2018.05.11/udpsender5.svg)](/media/svg/2018.05.11/udpsender5.svg)

This is where FreeBSD HEAD is at now.

It also turns out that we're partially receiver limited (on FreeBSD) fixing the inpcbhash lock bumps received packets to 4Mpps.

There is now a patch pending to change the inpcbinfo hash lock (the lookup table for inbound connections) from an rwlock to epoch for lookup and mutex for update.

This changes the profile to:
[![](/media/svg/2018.05.11/udpsender6.svg)](/media/svg/2018.05.11/udpsender6.svg)

At this point we're actually gated by the cxgbe driver's ability to push packets one at a time through the `if_transmit` interface. The card itself can push 40Mpps but we aren't going to get anywhere near that without pinning the transmitting threads to the CPU socket with the transmit queue that they hash to.





