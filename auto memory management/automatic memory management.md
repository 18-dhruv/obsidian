# It is one of the service provided by [[CLR]] provided during [[managed executions process]]. [[GC]] which is a part of CLR it is done by that




## Allocation of Memory 
when you initialize a process runtime (CLR) reserves a contigious region of memory for the new process .This reserved memory is called managed heap. Managed heap maintain a pointer  to the 
next memory address where the next object will be allocated,Initially this pointer is set to heap's
base address. All the reference types are allocated to managed heap. when an application creates - its first reference type it will be stored at base address and the next object to the next immidiate address.
 Allocating memory in managed heap is as fast as allocating memory at stack because  it is just adding a value to pointer. in addition object can be accessed very quickly because they are stored in contigious manner.

#