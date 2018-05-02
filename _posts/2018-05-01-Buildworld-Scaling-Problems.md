Sampling UNHALTED_CORE_CYCLES during buildkernel we see that there is a surprising amount of time spent in fstatat.
# `make -j96 buildkernel` 
![](/media/svg/2018.04.30/bk10.svg)

Moving on to buildworld we see that the overhead of fstatat is even worse.
# `make -j96 buildworld` 
![](/media/svg/2018.04.30/bw1.svg)

To isolate the issue I added various variations of fstatat calls to the microbenchmark suite, will-it-scale.
# `fstatat3_processes -t 96 -s 50`
![](/media/svg/2018.04.30/fstatat3.svg)

The problem occurs when we traverse a mountpoint in the path lookup, and need to reference the root vnode:
```
	/*
	 * Check to see if the vnode has been mounted on;
	 * if so find the root of the mounted filesystem.
	 */
	while (dp->v_type == VDIR && (mp = dp->v_mountedhere) &&
	       (cnp->cn_flags & NOCROSSMOUNT) == 0) {
		if (vfs_busy(mp, 0))
			continue;
		vput(dp);
		if (dp != ndp->ni_dvp)
			vput(ndp->ni_dvp);
		else
			vrele(ndp->ni_dvp);
		vrefact(vp_crossmp);
		ndp->ni_dvp = vp_crossmp;
		error = VFS_ROOT(mp, compute_cn_lkflags(mp, cnp->cn_lkflags,
		    cnp->cn_flags), &tdp);
		vfs_unbusy(mp);
		if (vn_lock(vp_crossmp, LK_SHARED | LK_NOWAIT))
			panic("vp_crossmp exclusively locked or reclaimed");
		if (error) {
			dpunlocked = 1;
			goto bad2;
		}
		ndp->ni_vp = dp = tdp;
	}
```

A possible solution is to keep a global table of references to mountpoint vnodes that are held until after a quiescent
period following an unmount.
