Before submitting substantial changes to FreeBSD one is expected to run `make universe` to compile kernel & world
for all architectures. This is intrinsically a bit of a slog based on the sheer amount of code one needs to compile.
However, as it turns out, one peculiarity of the design of bmake makes it far worse still:

```
last pid: 21086;  load averages: 179.49, 108.02, 50.89                                     up 0+09:31:50  09:52:34
2304 processes:202 running, 1925 sleeping, 177 waiting
CPU: 11.4% user,  0.0% nice, 88.6% system,  0.0% interrupt,  0.0% idle
Mem: 1910M Active, 58G Inact, 96G Wired, 30G Free
ARC: 76G Total, 35G MFU, 37G MRU, 26M Anon, 631M Header, 4418M Other
     64G Compressed, 152G Uncompressed, 2.39:1 Ratio
Swap: 2048M Total, 2048M Free
```

One asks ... what is using all this system time? Sampling `UNHALTED_CORE_CYCLES` yields the following:
![](/media/svg/2018.05.01/uni.svg)

This is an issue that Mateusz Guzik has raised a number of times in the past. Upon conferring with him I
learned that the following snippet in `contrib/bmake/job.c:Job_CatchOutput(void)` is the source of the
contention on the pipe mutex:

```
    /* The first fd in the list is the job token pipe */
    do {
	nready = poll(fds + 1 - wantToken, nfds - 1 + wantToken, POLL_MSEC);
    } while (nready < 0 && errno == EINTR);
```

Apparently the `make universe` process spawns hundreds of submakes all of which wait for events on the same "token pipe".
Unsurprisingly, this doesn't scale well. It's not clear at what scale this actually _does_ make sense. Perhaps on a
dual core getting a notification faster than simply sleeping for a millisecond or two to check for work is faster.


A complete universe build on a dual 8160 on a ZFS file system is just over 2 hours.

`make -j96 universe`:

```
--------------------------------------------------------------
>>> make universe completed on Wed May  2 15:21:02 PDT 2018
                      (started Wed May  2 13:24:06 PDT 2018)
--------------------------------------------------------------
```

A second universe build with no local modifications takes 11 minutes to simply check timestamps.
`make -s -j96 universe -DNO_CLEAN`
```
--------------------------------------------------------------
>>> make universe completed on Wed May  2 15:37:57 PDT 2018
                      (started Wed May  2 15:26:43 PDT 2018)
--------------------------------------------------------------
make -s -j96 universe -DNO_CLEAN  6391.96s user 55615.26s system 9203% cpu 11:13.74 total
```

Mateusz offered up the following patch to turn polling the pipe in to a sleep by way of a sleep:

```
diff --git a/contrib/bmake/job.c b/contrib/bmake/job.c
index c5eca9936efc..552438fb5189 100644
--- a/contrib/bmake/job.c
+++ b/contrib/bmake/job.c
@@ -2121,13 +2121,17 @@ Job_CatchOutput(void)
 {
     int nready;
     Job *job;
-    int i;
+    int i, pollToken;
 
     (void)fflush(stdout);
 
+       pollToken = wantToken;
+       if (getenv("BMAKE_NO_TOKEN_POLL") != NULL)
+               pollToken = 0;
+
     /* The first fd in the list is the job token pipe */
     do {
-       nready = poll(fds + 1 - wantToken, nfds - 1 + wantToken, POLL_MSEC);
+       nready = poll(fds + 1 - pollToken, nfds - 1 + pollToken, POLL_MSEC);
     } while (nready < 0 && errno == EINTR);
 
     if (nready < 0)
```

`BMAKE_NO_TOKEN_POLL=1 make -j96 universe -DNO_CLEAN`
```
--------------------------------------------------------------
>>> make universe completed on Wed May  2 15:49:44 PDT 2018
(started Wed May  2 15:41:58 PDT 2018)
--------------------------------------------------------------
BMAKE_NO_TOKEN_POLL=1 make -j96 universe -DNO_CLEAN  6478.89s user 35075.71s system 8913% cpu 7:46.22 total
```
We still see a large amount of system time:
```
last pid:  3484;  load averages: 89.14, 67.10, 78.69                                       up 0+15:24:24  15:45:08
1901 processes:193 running, 1531 sleeping, 177 waiting
CPU: 11.3% user,  0.0% nice, 88.4% system,  0.0% interrupt,  0.4% idle
Mem: 1021M Active, 74G Inact, 120M Laundry, 97G Wired, 14G Free
ARC: 83G Total, 38G MFU, 40G MRU, 2761K Anon, 793M Header, 4597M Other
     70G Compressed, 169G Uncompressed, 2.41:1 Ratio
Swap: 2048M Total, 2048M Free
```

But if we look at a profile of `UNHALTED_CORE_CYCLES` we see that the time is now spent predominately in VFS
operations.
![](/media/svg/2018.05.01/uni4.svg)
