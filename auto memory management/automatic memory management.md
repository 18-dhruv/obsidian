
# It is one of the service provided by [[CLR]] provided during [[managed executions process]]. [[GC]] which is a part of  CLR it is done by that




## Allocation of Memory 
when you initialize a process runtime (CLR) reserves a contigious region of memory for the new process .This reserved memory is called managed heap. Managed heap maintain a pointer  to the 
next memory address where the next object will be allocated,Initially this pointer is set to heap's
base address. All the reference types are allocated to managed heap. when an application creates - its first reference type it will be stored ati base address and the next obect to the next immidiate address.
 Allocatory in managed heap is as fast as allocating memory at stack because  it is just adding a value to pointer. in addition object can be accessed very quickly because they are stored in contigious manner.

# Releasing Memory 
when the garbage collector performs a collection, it releases memory for the object that are no longer be referenced by the application , it determines which object are no longer by examining the roots of application. Every application has set roots. Each root either refers to object or set to null .

### Roots 
Every application roots include static feilds, local variables and parameters on a thread stack , and CPU Register. GC has access to the list of active roots that are maintained by JIT and runtime, using this list, GC creates a graph that contain all the objects that are reachable from the root.

Object that are not in the graph , are considered unreachable by GC so it frees the memory allocated to them. During this collection the GC will identify the memory addesses occupied by the unreachable objects, and after the collection it uses a memory copying function to compact the reachable objects in memory, after this process GC do the pointer correction so that roots point to objects new location. It also sets the heap pointer after the last reachable object.
**collection only happens if GC find a significant no. of unreachable object**

To improve performance,gc allocates large objects to a separate part of heap (G2) 


![[IMG_20260723_110656.png]]