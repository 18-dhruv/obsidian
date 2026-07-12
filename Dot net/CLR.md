- [[CLR/CLI]]
- [[CLR/GC]]
- [[CLR/JIT]]
# **CLR** (Common language Runtime)
It is responsible for loading and execution of code writtein various Dot net programming lang 
- When C# program is compiled  , It is first converted into CLI (Comman intermediate Language) This is done by Language compilers in case of C Sharp It is CSE.==exe==
- This code can run on any platform that has CLR
- 
- when cli Program is executed it is first converted into ==machine - specific code by== ==CLR==
# Working of CLR :-
## Compilation and execution :-
 - when we write ==c sharp== code it get converted into  Cil
 - then CLR uses [[JIT]] to convert into machine specific code (It happens when program runs). 
## Services provided by CLR :-
- CLR handles automatic memory management through GC, prevents memory leaks 
- It ensures that data type is used safely and correctly 
- The CLR checks for CLI code for security risks before runing it 

## Cross-language integration :-
It allows different Dot net langs to work togather through Comman type system (CTS)

# key components:- 
- ## Managed code:- The cil code managed by CLR
- ## unmanaged code:- before .net There were technology which do not produced cil so that were managed by operating system
- ## Garbage collection([[GC]]) 
- [[JIT]]
- ## CTS (common type system) This ensures that data types accross different Languageare understood 