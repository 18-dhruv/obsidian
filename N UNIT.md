# Attributes 
- [TestFixtures] marks the class that it contains tests. This one doesnt matter modern nunit can detect it automatically
- [Test] works a single method as a test case. This is important because without this attribute Nunit wont know that its a test case and will not run it.
- [Setup] runs before every test in class. because of this test's doesnt effect each other 
- [TearDown] same idea as finally from our exception-handeling 
- [TestCase] runs same test with different inputs


