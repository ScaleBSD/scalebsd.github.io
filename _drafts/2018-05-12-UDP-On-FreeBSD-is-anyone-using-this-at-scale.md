
UDP performance on FreeBSD is bad. Not the quietly admit to your friends that it needs work bad. 
Nor even the "we don't talk abouth that in polite company bad. But rather the "drop it down a 
seismologically inactive mine, bury it under 50 tons of concrete, and then decree it a government
secret" bad.

Running 64 netperf clients `netperf -H $DUT -t UDP_STREAM -- -m 1` yields an activity profile like this:
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

Looks like:
[![](/media/svg/2018.05.11/udpsender2.svg)](/media/svg/2018.05.11/udpsender2.svg)

Straightforward to fix. But ...
