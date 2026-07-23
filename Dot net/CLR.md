

# **CLR** (Common language Runtime)
It is responsible for loading and execution of code writtein various Dot net programming lang 
- When C# program is compiled  , It is first converted into CLI.  ^first-mention
- (Comman intermediate Language) This is done by Language compilers in case of C Sharp It is CSE.exe 
- This code can run on any platform that has CLR
- when cil Program is executed it is first converted into ==machine - specific code by== ==CLR==
# Working of CLR :-
## Compilation and execution :-
 - when we write ==c sharp== code it get converte d into [[CLR#^first-mention|CLI]] and metadata( this metadata  tells  that what are the datatype what are there definations, to which member this data type reference)
 - then CLR uses JIT to convert into machine specific code (It happens when program runs). 
## Services provided by CLR :-
- CLR handles [[automatic memory management]] throug memory leaks 
- It ensures that data type is used safely and correctly 
- The CLR checks for CLI code for security risks before runing it 
## Cross-language integration :-
It allows different Dot net langs to work togather through Comman type system (CTS)

# key components:- 
- ## Managed code:- The cil code managed by CLR
- ## unmanaged code:- before .net There were technology which do not produced cil so that were managed by operating system
- GC
- ## CTS (common type system) This ensures that data types accross different Language are understood .
- ##  CLS (common Language specification) It defines common rules so that code written in different languages can interpolate
- ## value type:- value types will store the value directly into the memory location these types work with stack mechanism only der allows memory at compile tym
- ##  Reference type:-



# JVM VS CLR.                                                                
## 1. Intermidiate code.                     
Jvm-Bytecode.                   
Clr- CIL                      (Both  of them are Bytecod but CIL is for Multilanguage)

# 2.Type system 
JVM- single language system 
CLR - CTS and CLS (common language specification)
# 3. value type

JAVA-primitive types only 
Dot net- Primitive and custom type (you can create custom struct type)