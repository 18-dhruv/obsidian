 Hiding the implementation 


## solved problem by abstraction
suppose we are working on windows and we have to write a function that tells a printer to print but there is a challange . There are different cmpanies who make printer so every time a new company comes we have to change source code of windows But the concept behind the abstraction say that windows dont need to know the different companies it just needs Print method.

So we will just made a parent class printer and a function print().

## why did inheritance dont solves it 
by inheritance we will get the same function and the same implementations
But we expect different results from same method in this case print method. that's why virtual function exists 



## virtual function 
### there are two ways to resolve a method call 

 ![InkWriting](<Ink/Writing/2026.7.31 - 10.40am.svg>) [Edit Writing](https://youtu.be/2arL1jh8ihA?type=inkWriting&aspectRatio=3.810)


#### static Binding (Early Binding) - This is done on compile time
The compiler looks at the declare type of account which is BankAccount and it hard code this call to BankAccount  version into the cil but we wanted the InterestBankAccount
#### dynamic dispatch (Late binding) - this is done at runtime 
with the help of virtual keyword we can fix this, 
##### How? 
Because compiler doesn't knows in what the real object type is, it will be decided at runtime and will provide the implementation of the function that the type provides.

without dynamic dispatch [[Polymorphism]] is impossible


virtual function exists specifically because of generic code to still triggers type specific behaviour, whithout the code needing to know on check what the concrete type actually is. 

## How virtual function works under the hood
Every class with virtual function gets a hidden method table - essentially a array of function pointer.
every object of that class carries a hidden pointer (one table per type, shared by all instances of that type)
it virtual function call is compiled as "follow the object's pointer->looks up in the method table jumps to the address provided there. this cost a small performance cost


# new keyword 
its the opposite of the virtual keyword it deliberately do static binding.
Suppose a developer creates a function and don't use virtual in declaration so now u want use that method in derived class so u will use Keyword this will tell the Runtime that its a separate new method which doesn't have a relation with the base class method just the name is
## we have two ways to achieve abstraction 
- Abstract classes 
- Interface 


## Abstract class

can we make constructor Abstract ?no
only the members which are inherited can be abstract iii
 ![InkWriting](<Ink/Writing/2026.8.3 - 10.12am.svg>) [Edit Writing](https://youtu.be/2arL1jh8ihA?type=inkWriting&aspectRatio=5.333)

from this we know that every time we create a object for derived class first the fields that are inherited from the base class are initialized  first by calling the base class constructor then the derived class constructor is called.  
