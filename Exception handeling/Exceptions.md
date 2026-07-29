- In C#, errors in the program at runtime are propagated throught program by using a mechanism called exception

 ![InkWriting](<Ink/Writing/2026.7.28 - 14.26pm.svg>) [Edit Writing](https://youtu.be/2arL1jh8ihA?type=inkWriting&aspectRatio=5.333)




- If the class which is throwing the exception if it doesn't have a catch block the exception is passed to the method called the exception throwing error this is called propagating of the stack 
## Types of Exception 
- system exception - the exception thrown by CLR 
- application exception - These are custom exceptions defined by user
 ![InkWriting](<Ink/Writing/2026.7.28 - 22.09pm.svg>) [Edit Writing](https://youtu.be/2arL1jh8ihA?type=inkWriting&aspectRatio=5.333)



## properties of exception object 
- ### Message - when the exception occurs, the runtime makes a text message available to inform the user the nature of error and to suggest how to resolve it·
- ### stackTrace - the state of the stack when the exception was thrown. this trace list all the called methods and the line no. of source file where the methods are called·
 - ### InnerException - when u catch an exception we often want to throw custom exception that make more sense, so this property stores the original exception by










## throw vs throw e

- throw - it preserves the original stack trace 
- throw e- destroys the original stack trace it show the line where u wrote throw e instead of the