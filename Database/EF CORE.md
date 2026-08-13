
## Entity framework core 
It is a orm for .net
when u use EF core, you define C# classes and it translates the operation into corresponding sql query 


### core components 
- DbContext - It is a gateway to the database . This is a session manager, we create only one instance
- DBSet<class name>
- onconfigration override this method 