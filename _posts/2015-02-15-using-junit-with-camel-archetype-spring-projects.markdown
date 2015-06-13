---
layout: post
title:  "Using jUnit with camel-archetype-spring projects"
date:   2015-02-15 00:00:00
---
Recently I was looking around at ways of testing the Apache Camel assets I was developing. As a word of caution – I’m not actually a Java developer of any real kind, I’ve been in the .Net world for quite a few years and before that C then C++ on UNIX systems. That means you may find things in this with don’t quite fit the Java idioms, for example I am aware that I have a couple of constant strings in this which are not right – but going back and redoing them now I’ve noticed them isn’t high on my list of things to do. So, follow the high level goals but not necessarily the low level stuff if you know better.
The point behind this is to demonstrate how to do Test Driven Development (TDD) when developing using a camel-archetype-spring Maven project. This is really just a collection of thoughts and documenting what I have learnt along the way.

Creating the Project
--------------------
I’m using the Red Hat JBoss Developer Studio in this as the project was targeting JBoss Fuse in production.  This is based on Eclipse so most of this will be valid for using any Maven enabled development environment

![](http://blog-assets.systemidle.com/image-014.png)

The first step is to a new project using the Maven archetype.  Go through the new project steps to select a new Maven project and then on the Select an Archetype dialog select the camel-archetype-spring project.  Due to the target environment that I was using I had so un-tick the Show the last version of Archetype only option to be able to choose version 2.12.3 to match JBoss Fuse 6.2.

![](http://blog-assets.systemidle.com/image-015.png)

This does mean that it’s now almost three versions behind with Camel version 2.15 out very soon and so there will be things that are not available to you.  But then using the JBoss Fuse runtime environment does give your operations people a much richer environment to keep your business running and so as always with these types of decisions it’s important to listen to all the stakeholders.

I’ve called this project jUnitExample and once you have finished with the new project wizard a project structure will have been created that includes a camel-context.xml file and a couple of test files in the src/data directory.

The project is available as a [zip file](http://blog-assets.systemidle.com/jUnitExample.zip) with the files in their final state if you just want to get it running quickly.

The default camel-context file
------------------------------
The camel-context.xml file that is produced as part of the wizard includes a very simple route that moves a file from the src/data directory into one of two target directories depending upon the contents of the file.  If you open the file in the visual editor it appears as such.  All very pretty.  And also the last time I recommend that you actually open it like this.  Close it, right click on it and open as an xml file.  You have a choice, either totally develop using the visual editor or not use it at all.  Given that most of what I needed to do cannot be done through the visual editor easily (or at all) I just dumped it and went to the xml directly, probably too quickly but then I’m an old UNIX command line developer at heart.

![](http://blog-assets.systemidle.com/image-012.png)

The xml view gives you a fuller idea of what is going on without having to click around the icons and look at the properties.  You can see that it’s pulling from an uri with a file prefix.  The directory is relative to the project root.  The strange noop=true bit just tells it not to remove the file from the directory once it has processed it.  With the default options it keep track of which files it has processed and will only do them once (until you rerun camel that is.)

![](http://blog-assets.systemidle.com/image-013.png)

The code makes a choice – the when uses the first child node as the predicate and so in this case uses the xpath which tests the /person/city value.  If this is London then the message is logged as being in the UK and the the to node moves the message contents to a file.

The key is to understand that actions such as the from load in a message into the running context, known as the exchange and that further actions take place on this loaded message.  So it is passed into the to action which is how it has access to save it.  The exchange and the message within it flows down through the xml unto it gets to the end.  Note that the message flows through the to node as well, so it is still available after this point.

First Run
---------
But that’s enough explanation, let’s get the route running and doing its thing.  The simplest way to do this repeatedly is to set up a Run Configuration, so right click on the project and select Run As and then Run Configurations. Right click on the Maven Build option and select new.  Key things to do are to name it, set the Base directory (use Browse Workspace and select the project) and set the Goal to **camel:run**.  This is one of the targets that are preconfigures within this archetype.

![](http://blog-assets.systemidle.com/image-019.png)

Then Apply it, at which point it appears in the list and then Run it.

The console window will appear if it has been reduced and it will show a lot of things happening. At this first run it will be resolving all of the dependencies and fetching them from the repository using Maven.  Finally it will start up the camel route and hopefully show you two messages that come from your route – “UK message” and “Other message”.

![](http://blog-assets.systemidle.com/image-021.png)

All though this is such a simple route it already causes us trouble when it comes to testing it.  The problem is that it isn’t very testable – only way to run it is to actually run it through the camel:run Maven target. A decade ago we may have said we were done, but this isn’t anywhere near the definition of done (DoD) in most environments now.  So we need to get a unit test harness around it.

Getting back to TDD
-------------------
Well, that’s enough code without having a failing test, we’ve only just started and we’ve broken the [first of the three rules of TDD.](http://butunclebob.com/ArticleS.UncleBob.TheThreeRulesOfTdd)  To get back on track we need a failing test and given this is Java then the default choice (and one bundled already in the project) is jUnit.  

![](http://blog-assets.systemidle.com/image-023.png)

To create a test class there is a wizard which is accessed by right clicking on the src/test/java Source Project and selecting new Camel Test Case.  Browse for the Camel XML file under test and it will complete the rest with reasonable values.  Then select Next>, do not select Finish.

I’m sure the wizard is trying to be helpful in this next dialog in that it will write a log of code for you to pull results from the directories and other “helpful” stuff.

My advice is to un-tick all of them as you are just going to end up deleting most if not all of the code it writes for you anyway.  Again, wizards are helpful early on, but in this case we want to have much better control over the route being tested and there are other ways of doing this.

![](http://blog-assets.systemidle.com/image-027.png)

The wizard will have created a test class for you that very simply creates the context including the xml file with the route we want to test.  The @Test method itself doesn’t actually do anything yet.

However, let’s make sure we can run the test and make it pass, then at least we know we are getting somewhere.  Right click on the project and select Run as, Maven Test and it will unsurprisingly result in one test run, build success.  Again, we haven’t really tested anything – nothing was being asserted within the test case.

Mocking Framework
-----------------
So let’s make the test fail creating demand for more production code. At this point let’s introduce the Mock framework which is part of the camel-core package.  Testing is such a central part of camel that the mocking framework is part of the core package and so is already available.  The test class is derived from CamelSpringTestSupport which includes a wide range of very useful methods.

![](http://blog-assets.systemidle.com/image-030.png)

The MockEndpoint class allows the test class to set expectations and assert against them, for example in this situation we are expecting that the mock:fileUK endpoint will receive the fileUK string when we send that as the body to the direct:files endpoint.

To be able to determine the data going through the test and not have it become brittle due to changing data outside of the test case I include the contents of the two test xml files as strings within the class.

It will come as no surprise that when we run this test it will result in a failure saying that there were no consumers available on the Image 029direct:files endpoint.  The direct: endpoint is synchronous, you can only put a message onto it if there is exactly one route listening to it.  Other endpoints will act as a queue isolating senders from this situation, but direct has no queue capability.  And in our case this is a good thing as it’s now forced the test case to fail.  This gives us a pull situation so we can move back to our production code.

![](http://blog-assets.systemidle.com/image-032.png)

The simplest way of fixing this issue is to change the camel-context.xml file to use these endpoint names.

Now when we run Maven test it all passes without issue.  But of course, we now have a route that doesn’t actually work outside of the unit test harness, so not very useful.  To sort this issue out we need to parameterise the endpoints so that they can be the original values when using camel:run and the test values when running jUnit.  This is possible which a few changes.

Using Parameters
----------------
First we have to instruct the route to use parameters, this is done by adding the parameters bean to the route xml file before the <camelContext> section starts. 

![](http://blog-assets.systemidle.com/image-037.png)

This tells it that there is a file called placeholder.properties that contains property names and values that are needed.  So we need to create this file.  Right click on the src/main/resources Source Folder and select New -> Other, search for properties and select the property file option.

![](http://blog-assets.systemidle.com/image-033.png)

Now when you run as camel:run it will once again work – but our test harness will fail as all this has done is revert the values back to the original values all be it through a properties file.

Using Multiple Parameter Files
------------------------------
The trick is that we can override the property file location in our tests.  So create another property file this time under src/test/resources calling it placeholder.UnitTesting.properties.

![](http://blog-assets.systemidle.com/image-034.png)

So that this file will be used when we run the test harness we need a properties bean that includes this new properties file.  To do this we need to create a new xml file in the src/test/resources Source Folder called UnitTestPropertiesOverride.xml with the contents as shown.

![](http://blog-assets.systemidle.com/image-035.png)

As the bean id in this file is the same as the one in the camel-context.xml file it will override that original definition – and in doing so the UnitTesting properties file will be pulled in over the main properties file.

![](http://blog-assets.systemidle.com/image-036.png)

To actually instruct the test harness to perform this slight of hand we need to modify the test class in the method that loads the camel-context.xml file – the createApplicationContext.  The two xml files will be applied in order thus repointing the unit tests to the unit testing property file

Now when you run the unit tests (Maven test) they will pass. 

![](http://blog-assets.systemidle.com/image-038.png)

Add in another test method for the “other” option and you now have two tests for the two possible ways through the route.

This has resulted in the route being testable.  We set our expectations of the route and determine its input so that the test is controlled and no longer brittle due to file contents changing.
