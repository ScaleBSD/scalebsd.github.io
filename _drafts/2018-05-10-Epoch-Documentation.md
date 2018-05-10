
Benefits:
+ epoch does not have lock ordering issues
+ epoch_enter will never block

Limitations:
- While in an epoch section one can't do a preemptible epoch_enter on a different epoch_t
- No sleeping or sx lock acquisition in an epoch section
- One must use safe list traversal and removal routines
- Object destruction must follow a grace period
