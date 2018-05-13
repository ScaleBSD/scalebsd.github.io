
UDP performance on FreeBSD is bad. Not the quietly admit to your friends that "it needs work" bad. 
Nor even the "we don't talk abouth that in polite company" bad. But rather the "drop it down a 
seismologically inactive mine, bury it under 50 tons of concrete, and then decree it a government
secret" bad.

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
