T is called generic type 
It is basically a placeholder that get substituted with the actual type when u use the class, not when you define it.
## Difference from java
- In Java Generics are compile time only illusions List. Integer and List is same at runtime

 ![InkWriting](<Ink/Writing/2026.8.11 - 11.02am (2).svg>) [Edit Writing](https://youtu.be/2arL1jh8ihA?type=inkWriting&aspectRatio=5.333)

Here the compiler silently add a cast
- so when we use .get function it return object type but because compiler add a cast we get the expected return type
But this also the reason we can't check the return type  at compile time in java



# in case of .net
when u write generic type, the compiler don't fully resolve it generates generic cil with placeholder.   
 At each concrete instantiation CLR decides whether its a value type or a reference type both are handled differently
 ## value type                                                                                                                                                   Node int  and Node double both get different machine code
 This is why List int in C# is fast the underlying array is real int [] not an object [] full of boxed integer 

   ## Reference type                
   


