## the managed execution process includes the following steps :-
## choosing a compiler :- 
 to obtain the benifit by [[CLR]] you must use one or more language compiler 
 that target the runtime, because its a multilanguage execution
## Compile [[CLI]] to native code :-
Before you can run cil, it must be compiled against the CRE to native code for the target machine architecture
### there are two ways to perform this conversion 
- [[JIT]]
- [[Ngen.exe(Native image Generator)]] (this is used only in classic .net )
- when it is rewriten to cross platform then they replaced it with [[crossgen]](it is the direct successor of negen.exe) 
- [[Native Aot]]

both jit and and other aot are used in diffrent senarios when u have to ship the final app to the user u will use Aot .
and if your app's il changes constantly then u will have to use jit.

## 3. code verification :-
as a part of compilation to native code , the CIL code must also pass a verification process . in this process CIL and metadata  are verified to find out whether the code is type safe or not  , and also examined for  in an attempt to confirm that the code  can access memory location and call methods only through proper defined types .

## 4. run code :-
before a method can be run , it must be compiled to processor-specific code. each method can be run , it nust be compiled to processor specific code.








