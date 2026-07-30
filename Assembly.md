## An assembly is a collection of types and resources that are built to work together and form a logical unit of functionality.

The two forms of Assembly are 
- DLL (Dynamic link library) 
- exe(executable)


Both of them have identical structure (CIL + metadata + manifest) but both of them have different purpose. 
- #### exe
   - It has a Entry point the OS/CLR calls to start execution (main)
   - It represent standalone application
   - only one main is allowed 

- #### dll
   - No entry point 
  - contain reusable types, methods, classes meant to be refered by other code


## GAC(global assembly cashe) legacy
each computer that has CLR have machine wide code cashe called GAL. It stores assemblies specification designated to shared by several application on computer
 
 ### there are two way to install application to GAC
 - use a installer to designate to work with GAC. this is preffered 
 - GAC tool provided by windows SDK
 - we can obtain programmatically information about an assembly using [[reflection]]
### GAC is legacy now the problem with GAC was suppose I have two  apps they defend on same assembly we want to update dependencies for one app but because of that the second app breaks because it defends on that to solve três problem we use NuGet 

## NuGeT - flips the model instead of using the global copy, each project declare exactly which version it needs
