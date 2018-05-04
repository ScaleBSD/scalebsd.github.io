
```
diff --git a/sys/vm/vm_map.c b/sys/vm/vm_map.c
index 1ce2435fe3c..db58cea18e4 100644
--- a/sys/vm/vm_map.c
+++ b/sys/vm/vm_map.c
@@ -3157,8 +3157,9 @@ vm_map_delete(vm_map_t map, vm_offset_t start, vm_offset_t end)
                if (entry->wired_count != 0) {
                        vm_map_entry_unwire(map, entry);
                }
-
-               pmap_remove(map->pmap, entry->start, entry->end);
+               if ((entry->eflags & MAP_ENTRY_IS_SUB_MAP) != 0 ||
+                       entry->object.vm_object != NULL)
+                       pmap_remove(map->pmap, entry->start, entry->end);
 
                /*
                 * Delete the entry only after removing all pmap
```

```
mmacy@anarchy [~/devel/will-it-scale|22:08|38] ./mmap1_processes -t 96 -s 5 
Before:
min:7615 max:12599 total:1009155
average:1008730
After:
min:7827 max:13122 total:1025163
average:1029671
```
[![](/media/svg/2018.05.03/mmap1_master_vmfix.svg)](/media/svg/2018.05.03/mmap1_master_vmfix.svg)


```
void testcase(unsigned long long *iterations, unsigned long nr)
{
	unsigned long pgsize = getpagesize();

	while (1) {
		unsigned long i;

		for (i = 0; i < MEMSIZE; i += pgsize) {
			char *c = mmap(NULL, MEMSIZE, PROT_READ|PROT_WRITE,
						   MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
			assert(c != MAP_FAILED);
			c[i] = 0;
			(*iterations)++;
			munmap(c, MEMSIZE);
		}

	}
}
```

```
mmacy@anarchy [~/devel/will-it-scale|21:58|25] ./page_fault4_processes -t 96 -s 5
HEAD:
min:7445 max:12675 total:930033
average:928180
EBR:
min:9186 max:17250 total:1186554
average:1176056
```

HEAD w/ reserve locking fixed
[![](/media/svg/2018.05.03/pf4_master_vmfix.svg)](/media/svg/2018.05.03/pf4_master_vmfix.svg)

EBR w/ reserve locking fixed
[![](/media/svg/2018.05.03/pf4_epoch.svg)](/media/svg/2018.05.03/pf4_epoch.svg)


<--

mmacy@anarchy [~/devel/will-it-scale|22:09|39] ./page_fault1_processes -t 96 -s 5
testcase:Anonymous memory page fault
warmup
min:79557 max:184561 total:11931525
min:65536 max:150563 total:9985428
min:65536 max:157213 total:10648464
min:66982 max:147379 total:9837071
min:65536 max:150025 total:9731626
min:64172 max:145891 total:9863466
measurement
min:66900 max:151808 total:9778821
min:66718 max:158599 total:10074983
min:71381 max:153899 total:10558106
min:62706 max:153520 total:9746283
min:47396 max:145235 total:9736057
average:9978850
pf1_master_vmfix.svg



mmap1_epoch.svg pf1_epoch.svg   pf4_epoch.svg
mmacy@anarchy [~/devel/will-it-scale|22:30|2] ./page_fault1_processes -t 96 -s 5
testcase:Anonymous memory page fault
warmup
min:93302 max:163840 total:12036728
min:67438 max:146947 total:9548865
min:69477 max:136065 total:9890874
min:65536 max:134743 total:9833380
min:67507 max:139754 total:10308253
min:71836 max:148775 total:10801110
measurement
min:71847 max:150920 total:10469724
min:70106 max:137250 total:9854826
min:78087 max:153486 total:11192901
min:64901 max:140737 total:9691438
min:65536 max:134462 total:9736995
average:10189176

-->
