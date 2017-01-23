# Java Threads and Locks


### Readings

* Java Language Specification
  * [Java Memory Model](https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html#jls-17.4)
* StackOverflow
  * [Relationship of reordering & happens-before in java](http://stackoverflow.com/questions/16213443/instruction-reordering-happens-before-relationship-in-java)
  * [Why setArray() method call required in CopyOnWriteArrayList](http://stackoverflow.com/questions/28772539/why-setarray-method-call-required-in-copyonwritearraylist)
  * [Java Compiler Reordering](http://stackoverflow.com/questions/31015169/java-compiler-reordering)
* Wiki
  * [Memory model](https://en.wikipedia.org/wiki/Memory_model_(programming)
  * [Memory ordering](https://en.wikipedia.org/wiki/Memory_ordering)
  * [Memory barrier](https://en.wikipedia.org/wiki/Memory_barrier)
  * [Out of Order Execution](https://en.wikipedia.org/wiki/Out-of-order_execution)
* [Happens-Before俗解](http://ifeve.com/easy-happens-before/)
* [Understanding JIT](http://docs.oracle.com/cd/E15289_01/doc.40/e15058/underst_jit.htm)
* Video
  * [Java Memory Model](https://www.youtube.com/watch?v=WTVooKLLVT8)


### Memory Reodering
Several mechanisms can produce the reordering. A **Just-In-Time**(JIT) compiler in a Java Virtual Machine implementation may rearrange code, or the **processor**. In addition, the memory hierarchy of the architecture on which a Java Virtual Machine implementation is run may make it appear as if code is being reordered. 
* Compiler can reorder statements
* Processor can reorder them
* On multi-processor, values not synchronized to global memory
* The memeory model is designed to allow aggressive optimization
* Good for performance
