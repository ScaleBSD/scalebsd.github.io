Where do we go from here? Benchmarks can identify how well a system performs
but are specific to one workload and configuration. Microbenchmarks are useful 
for identifying system bottlenecks but they're clearly too ad hoc in their coverage. 
Is there a way to more consistently identify issues? Up until recently the answer was 
no. However, work in 2014 by Clements [Clements, 2014] built on the notion of 
disjoint-access-parallel memory systems [Israeli, 94] to rigorously identify 
limitations in the scalability of both software interfaces and their implementations.
This work proposes the _scalable commutativity rule_, which says, in essence,
that whenever interface operations  commute, they can be implemented in a way
that scales. The intuition behind this is simple: when operations commute, their 
results (both the values returned and any side effects) are independent of order. 

He starts by observing that many scalability problems lie not in the implementation, 
but in the design of the software interface. An interface definition that does not 
permit two operations to commute enforces serialization between two calls. The
POSIX definition of the _open_ system call requires that it return the lowest available
file descriptor. This means that two calls to open of different files need to be serialized
on file descriptor allocation. Some other system calls that have unnecessarily unscalable 
interfaces are: fork (when immediately followed by exec), stat, sigpending, and munmap.

This is an interesting observation, but the real contribution of the work is developing a
tool called _COMMUTER_ which:
 1) takes a symbolic model of an interface and computes precise conditions  for  when  
     that  interface’s  operations  commute.  
 2) uses these conditions to generate concrete tests of sets of operations that commute 
     according to the  interface  model,  and  thus  should  have  a  conflict-free  
     implementation  according  to  the  commutativity  rule. 
 3) checks  whether  a  particular  implementation  is  conflict-free for each test case. 

He applied this to 18 POSIX system calls to generate 26, 238 test cases and used 
these to compare Linux with sv6, a research OS developed by his group. He found that
on Linux 17,206 cases scale vs 26,115 on sv6. The collection of test cases that failed 
to scale can be used as a starting point for redesigning subsystems just as the 
will-it-scale benchmarks have enabled us to identify a much narrower set of issues.

Porting COMMUTER to work with FreeBSD would be an interesting avenue for future work.
