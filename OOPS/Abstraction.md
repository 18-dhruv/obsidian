 Hiding the implementation 


## solved problem by abstraction
suppose we are working on windows and we have to write a function that tells a printer to print but there is a challange . There are different companies who make printer so every time a new company comes we have to change source code of windows But the concept behind the abstraction say that windows dont need to know the different companies it just needs Print method.

So we will just made a parent class printer and a function print().

## why did inheritance solves it 
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


virtual function exists 