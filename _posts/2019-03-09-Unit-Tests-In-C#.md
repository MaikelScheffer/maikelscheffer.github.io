---
layout: post
title:  "Taking the pain away: A hater's guide to Unit Tests in C#"
date:   2019-03-09 13:33:40 +0100
categories: C#, .NET, Unit Testing
---

### How to write unit tests and not want to die (more than usual)

You read about it everywhere: unit tests.  
People praise it to high heavens and consider it God's gift to mankind.   
In this guide I'll show you how you can actually get running and running with unit tests. 

That's right, in this guide I won't get into "getting up and running", but just the running.  
There's plenty of guides out there that'll tell how unit tests work and about the TDD feedback loop and all that. 

No, this guide is for those among us that understand the importance of unit tests, the gain they could provide, but nevertheless have never really been able to make it part of your workflow.

In this i'll be using:

* C#
* xUnit 2
* EntityFrameworkCore (very briefly, before we start faking it)


#### Content

* The common pure pain example
* Injecting morphine: AutoFixture 
* Chugging down painkillers: MOQ
* By their powers combined: ~~Crack~~ AutoMoq
* I'm being abused by my relational DB and its circular references


### Pain (Without love)

In this example we'll be using nothing but xUnit, Entityframework core, and C# itself.

{% highlight c# %}

[Fact]
public void GetHeadphones_Get_ShouldPass()
{
    // Starting with the basic test method
    // Methodname consists of Method name, case under test, expected results
}

{% endhighlight %}

So, we want to retrieve a bunch of `Headphones` from  the database.
We have a databsase running somewhere that has a table filled a list of headphones.




{% highlight c# %}

[Fact]
public void Headphone_GetHeadphones_ShouldPass()
{
    // database with 2 rows
    // pretend we have no  OnModelCreation() filling the connection string
    var dbOptions = new DbContextOptionsBuilder<HeadphonesContext>()
                    .UseSqlServer(Config.GetConnectionString())
                    .Options;

    var context = new HeadphonesContext(dbOptions);

    IHeadphoneRepo repo = new HeadphoneRepo(context);

    var service = new HeadphoneService(repo);

    var headphones = service.GetHeadphones();

    Assert.Equal(2, headphones.Count());
}

{% endhighlight %}

Thing is that we can't just query the DB, that would make our unit tests dependant on an outside resource. You heathen.  
The connection could time out, throw back a sql deadlock, or anything else outside of our control.  
But most importantly, connecting to the database takes time and we need our unit tests to be fast.   

But more, more importantly: the database is mutatable.  
Your test could expect getting 3 headphones from the database, but then some intern gone and decided to delete a row from the database.
"Oh, no. My precious test cases" you scream, stomping over to berate an intern into an accounting career. 

I mean, you could setup a test DB and use your unit tests against that.   
If bashing a database into submission is your kinda thing, by all means.

Entityframeworkcore to the rescue: In-memory DB

{% highlight c# %}

    [Fact]
    public void Headphone_GetHeadphones_ShouldPass()
    {
        var dbOptions = new DbContextOptionsBuilder<HeadphonesContext>()
                        .UseInMemoryDatabase(databaseName: "UnitTestDB")
                        .Options;

        var context = new HeadphonesContext(dbOptions);

        IHeadphoneRepo repo = new HeadphoneRepo(context);

        var service = new HeadphoneService(repo);

        var headphones = service.GetHeadphones();

        Assert.Equal(2, headphones.Count());
    }

{% endhighlight %}

Wow, amazing. Move it into a seperate class.  
Call it from the constructor on your test.  

{% highlight c# %}

private readonly HeadphonesContext _context;

HeadphoneTests() 
{
    _context = Setup.GetContextUsingMemoryDB();
}

[Fact]
public void Headphone_GetHeadphones_ShouldPass()
{
    IHeadphoneRepo repo = new HeadphoneRepo(_context);

    var service = new HeadphoneService(repo);

    var headphones = service.GetHeadphones();

    Assert.Equal(2, headphones.Count());
}

{% endhighlight %}

There's just one problem though.    
That memoryDB doesn't persist data, being memory and all.    
Well, dang. Better create some data then?

{% highlight c# %}

[Fact]
public void Headphone_GetHeadphones_ShouldPass()
{
    var headphonesOne = new Headphone
    {
        Id = 1,
        Name = "Bose QuietComfort 35",
        Type = HeadPhoneType.NOISE_CANCELLING,
        Version = "1.0.0"
    };

    var headphonesTwo = new Headphone
    {
        Id = 2,
        Name = "AudiaTechnica ACK35",
        Type = HeadPhoneType.ON_EAR,
        Version = "2.0.0"
    };

    var listOfHeadphones = new List<Headphone>() 
    { 
        headphonesOne,
        headphonesTwo 
    };

    context.Headphones.AddRange(listOfHeadphones);
    context.SaveChanges();

    IHeadphoneRepo repo = new HeadphoneRepo(_context);

    var service = new HeadphoneService(repo);

    var headphones = service.GetHeadphones();

    Assert.Equal(2, headphones.Count());
}

{% endhighlight %}

Maybe move that object creation bit in a seperate method?   
Call it something like `MockData.CreateHeadphonesInMemoryDB()`? 

Hold on, why does the context suddenly have 4 headphones?

Welcome to the realm of the damned.     
The constant creation of new objects that sorta match test cases.    
Pushing them into a test db or memory db and reconsidering opening a bakery.     
This is about the point where most people quit unit tests.   

### Using Autofixture

Turns out there's several NuGet packages out there that'll create that garbage for you.

{% highlight c# %}

[Fact]
public void Headphone_GetHeadphones_ShouldPass()
{
    var items = _fixture.CreateMany<Headphone>(2).ToList();

    context.Headphones.AddRange(listOfHeadphones);
    context.SaveChanges();

    IHeadphoneRepo repo = new HeadphoneRepo(_context);

    var service = new HeadphoneService(repo);

    var headphones = service.GetHeadphones();

    Assert.Equal(2, headphones.Count());
}

{% endhighlight %}


{% highlight c# %}

[Fact]
public void Headphone_GetHeadphones_ShouldPass()
{
    var items = _fixture.CreateMany<Headphone>(2).ToList();

    var mock = new Mock<IHeadphoneRepo>();
    mock.Setup(x => x.GetHeadphones()).Returns(items);

    var service = new HeadphoneService(mock.Object);

    var headphones = service.GetHeadphones();

    Assert.Equal(2, headphones.Count());
}

{% endhighlight %}