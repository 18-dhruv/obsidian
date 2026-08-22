
## Entity framework core 
It is a orm for .net
when u use EF core, you define C# classes and it translates the operation into corresponding sql query 


### core components 
- DbContext -
 ![InkWriting](<Ink/Writing/2026.8.21 - 20.00pm.svg>) [Edit Writing](https://youtu.be/2arL1jh8ihA?type=inkWriting&aspectRatio=5.333)
 It is a gateway to the database . This is a sessional manager, we create only one instance
- DbSet class name (this is the representation of a table in C#)

 ![InkWriting](<Ink/Writing/2026.8.21 - 20.03pm.svg>) [Edit Writing](https://youtu.be/2arL1jh8ihA?type=inkWriting&aspectRatio=5.333)

- onconfigration override this method 