---
layout: post
title: Testing an API In Memory in ASP.NET Core
date: 2019-01-28 16:08:25
author: dgiddins
tags: [Testing, DotNetCore, ASP.Net, NUnit]
---

## TL;DR
This is long post that describes how to setup an in-memory test harness for testing an entire ASP.NET Core API with lots of code examples. 

## Testing and broccoli

Unit testing, like broccoli, is something we all know we should do, but actually we just want to tuck into the meat and potatoes of production code. One reason you might resist writing unit tests is that you are writing an API which is mostly a CRUD layer over a database (where most of the code is right-to-left copying from a DTO to an API response), or you are writing framework plugins for which you just can't write meaningful tests (for instance ActionFilters or Binder). 

Well I understand, I've been in the same situation, but I have a way of dealing with this problem. What I prefer to do is test at the level of http all the way through the API to the database. But wait, you may cry, that's not a unit test! Well yes you have a point there, I'm touching lots of different classes when I test like this. I would argue however that the word "unit" is a bit ambiguous; my definition of unit is not 'a single class'. 

I prefer to classify tests as either **Isolated**, **Integration**, or **End-to-End**. When I say isolated, what I mean is that the test can be run without worrying about external dependencies, whether they are your databases or an external API or other system. This means you can test anything from a single class up to the whole API, reliably and repeatably. Integration tests then focus on integrations with your database, external APIs etc. These tests are simple and limited in scope so when they break your integration code is broken or the system is down. End-to-End tests end up being more akin to smoke tests or sanity tests and any break in these should be reflected in your Isolated and Integration tests. 

Given these definitions let's look at how you can test your API end-to-end in an isolated fashion. Or put another way, let's cook that broccoli in butter and chilli and season with salt, pepper and a little grated parmesan (you're welcome!)

In order to test an ASP.NET Core application we need to be able to spin up our whole application in a test harness (i.e. your unit test/test fixture). Microsoft provide a NuGet package that lets us do just that; [Microsoft.AspNetCore.TestHost](https://www.nuget.org/packages/Microsoft.AspNetCore.TestHost/) contains a class called TestServer. Once you create an instance of the TestServer you can get an instance of a HttpClient with which you can then make http calls to your API. 

## Creating a TestServer instance

To create an instance of the TestServer you need to supply its constructor with an instance of a WebHostBuilder. The easiest way to create a WebHostBuilder is in the same manner as you would when you are building your IWebHost implementation to run your site with Kestrel.

```csharp

    public class Program
    {
        public static void Main(string[] args)
        {
            BuildWebHost(args).Run();
        }

        private static IWebHost BuildWebHost(string[] args)
        {
            return new WebHostBuilder().UseKestrel()
                .UseContentRoot(Directory.GetCurrentDirectory())
                .UseStartup<Startup>()
                .UseUrls("http://*:" + ApplicationInfoProvider.GetPort())
                .Build();
        }
    }

```

So in a test harness you would write

```csharp

  var webHostBuilder = new WebHostBuilder().UseStartUp<Startup>());
  var server = new TestServer(webHostBuilder);
  var httpClient = server.CreateClient();

```

The problem with this is if you just use your Startup class that you use for normally spinning up your site, you end up with the exact same configuration and therefore the same dependencies. You may think to create a new version of Startup (InMemoryStartup for instance) and reference that; the problem then is that you don't have an easy way to manipulate the state of your API from your test harness, unless you want to use static properties. That might work in simple cases, but with complex systems managing that static state might become problematic - and who likes statics anyway. 

What we really want to do is pass on instance of a Startup class (designed for in-memory testing) to the WebHostBuilder constructor, but there is no native support for this. To get around this use the following extension method.

```csharp

    public static class WebHostBuilderExtensions
    {
        public static IWebHostBuilder UseStartupInstance(this IWebHostBuilder hostBuilder, IStartup startup)
        {
            string name = startup.GetType().GetTypeInfo().Assembly.GetName().Name;
            return hostBuilder.UseSetting(WebHostDefaults.ApplicationKey, name).ConfigureServices((services =>
            {
                if (typeof(IStartup).GetTypeInfo().IsAssignableFrom(startup.GetType().GetTypeInfo()))
                    services.AddSingleton(startup);
                else
                    services.AddSingleton(typeof(IStartup), serviceProvider =>
                    {
                        IHostingEnvironment requiredService = serviceProvider.GetRequiredService<IHostingEnvironment>();
                        return new ConventionBasedStartup(StartupLoader.LoadMethods(serviceProvider, startup.GetType(), requiredService.EnvironmentName));
                    });
            }));
        }
    }

```

> As an aside, this post https://www.stevejgordon.co.uk/aspnet-core-anatomy-how-does-usestartup-work is a very in depth look at how Startup classes work in Asp.net.

You might have noticed the IStartUp type in the UseStartupInstance above and be thinking, wait, my Startup class doesn't implement that, what's going on? Well, it's basically some convention-based magic which is explained and examined in the blog post I reference above. It's the kind of thing that personally irritates me as it makes things less discoverable for the sake of not writing ``" : IStartup"`` for each project (This isn't really why its done but thats what it feels like). 

We can now inject a Startup class as shown below.

```csharp

  var webHostBuilder = new WebHostBuilder().UseStartupInstance(new Startup());
  var server = new TestServer(webHostBuilder);
  var httpClient = server.CreateClient();

```

But now this won't work because your normal Startup class does not implement IStartup, therefore we need to create an InMemoryStartup class that does implement this, and which we can customize for testing purposes. This would look something like the following.

```csharp

    public class InMemoryStartup : IStartup
    {
        public InMemoryStartup()
        {
        }

        public IServiceProvider ConfigureServices(IServiceCollection services)
        {
            // Some code to configure services
        }

        public void Configure(IApplicationBuilder app)
        {
            // Some code to configure application
        }
    }

```

You would then construct your test harness as follows (if you are using NUnit)

```csharp

[TextFixture]
public class SomeInMemoryTests
{
  private HttpClient _httpClient;

  [Setup]
  public void StartupInMemoryApi()
  {
    var webHostBuilder = new WebHostBuilder().UseStartupInstance(new InMemoryStartup());
    var server = new TestServer(webHostBuilder);
    _httpClient = server.CreateClient();
  }

  [Test]
  public async Task TestGettingSomeData()
  {
    var response = await _httpClient.GetAsync("/some-data/123");
    var data = JsonConvert.DeserializeObject<Data>(await response.Content.ReadAsStringAsync());
    Assert.That(data.Id, Is.EqualTo(123));
  }

  [Test]
  public async Task TestPostingSomeData()
  {
      //etc...
  }
}

```

As you can see this provides a simple method of testing an API inside your test harness. The HttpClient provides convenience methods for GET, POST, PUT PATCH and DELETE and also a general SendAsync method that allows for fine grained control. 

What I've described here isn't much more interesting or detailed than what is already described in similar posts. And like all of them so far we have skipped over an important detail; what about databases or any other external service on which I depend? If I simply replicate my application startup logic, I'm still pointing at a real database. This will prevent the tests from being repeatable and reliable as the external database is outside the control of the test harness.

To get around this we will need to design our InMemoryStartup class to inject Test Doubles (you might refer to them as Mocks, see [here](https://martinfowler.com/bliki/TestDouble.html) for definitions) into the application in place of your abstractions around your external dependencies (and I assume you are doing this right).

The place to do this is in the ConfigureServices method where we are setting up our dependency injection container with our dependencies. We would also want to be able to pass in our Test Doubles from our test harness so that the test can control these doubles and also read data from them. The InMemoryStartup class would end up looking something like this.

```csharp

    public class InMemoryStartup : IStartup
    {
        private IDatabase _databaseTestDouble;
        private IClientApi _apiTestDouble;
        private ILog _logTestDouble;

        public InMemoryStartup(IDatabase databaseTestDouble, IClientApi apiTestDouble, ILog logTestDouble)
        {
            _databaseTestDouble = databaseTestDouble;
            _apiTestDouble = apiTestDouble;
            _logTestDouble = logTestDouble;
        }

        public IServiceProvider ConfigureServices(IServiceCollection services)
        {
            services.AddSingleton<IDatabase>(_databaseTestDouble);
            services.AddSingleton<IClientApi>(_apiTestDouble);
            services.AddSingleton<ILog>(_logTestDouble);

            //Add other dependencies that are not external
        }

        public void Configure(IApplicationBuilder app)
        {
            // Some code to configure application
        }
    }

```

Given this we can now modify our tests as follows...

```csharp

[TextFixture]
public class SomeInMemoryTests
{
  private HttpClient _httpClient;
  private Data _testData;

  [Setup]
  public void StatupInMemoryApi()
  {
    _testData = Data() {Id = 123, Data = "some data"};
    var databaseDouble = new FakeDatabase() { Data = new List<Data>(){testData}};
    var startUp = new InMemoryStartup(databaseDouble, null, null);
    var webHostBuilder = new WebHostBuilder().UseStartupInstance(startUp);
    var server = new TestServer(webHostBuilder);
    _httpClient = server.CreateClient();
  }

  [Test]
  public async Task TestGettingSomeData()
  {
    var response = await _httpClient.GetAsync("/some-data/123");
    var data = JsonConvert.DeserializeObject<Data>(await response.Content.ReadAsStringAsync());
    Assert.That(data.Id, Is.EqualTo(_testData.Id));
    Assert.That(data.Data, Is.EqualTo(_testData.Data));
  }
}

```

In this particular example I have only supplied a test double for the database, in other tests you would want to provide doubles for the other dependencies as well. In fact I would suggest that you should set-up all of your dependencies for every test case and do so consistently for all tests. I would do this for the simple reason that you may suffer side effects when the different tests run in different setups of your API. 

Given that it is advisable to set-up all of your dependencies for every tests it becomes obvious that you should encapsulate the setup of your dependencies in a class that you might call InMemoryApi. This could look something like this.

```csharp

public class InMemoryApi
{
    public InMemoryApi()
    {
        DatabaseTestDouble = new FakeDatabase();
        ClientApiTestDouble = new FakeClientApi();
        LoggerTestDouble = new FakeLogger();

        var startUp = new InMemoryStartup(DatabaseTestDouble, ClientApiTestDouble, LoggerTestDouble);
        var webHostBuilder = new WebHostBuilder().UseStartupInstance(startUp);
        var server = new TestServer(webHostBuilder);
        Client = server.CreateClient();
    }

    public FakeDatabase DatabaseTestDouble { get; private set; }
    public FakeClientApi ClientApiTestDouble { get; private set; }
    public FakeLogger LoggerTestDouble { get; private set; }
    public HttpClient Client {get; private set; }
}

```

Which then simplifies your test harness to look like this:

```csharp

[TextFixture]
public class SomeInMemoryTests
{
  private InMemoryApi _api;
  private Data _testData;

  [Setup]
  public void StartupInMemoryApi()
  {
    _testData = Data() {Id = 123, Data = "some data"};
    _api = new InMemoryApi();
    _api.FakeDatabase.AddTestData(_testData); //a method on your test double
  }

  [Test]
  public async Task TestGettingSomeData()
  {
    var response = await _api.Client.GetAsync("/some-data/123");
    var data = JsonConvert.DeserializeObject<Data>(await response.Content.ReadAsStringAsync());
    Assert.That(data.Id, Is.EqualTo(_testData.Id));
    Assert.That(data.Data, Is.EqualTo(_testData.Data));
  }
}

```

You can now easily write further tests, reusing this InMemoryApi type with very little additional effort. All of your usual techniques for creating a common test context can be used to further reduce duplicate setup code. You can also use libraries such as [NMock](http://nmock.sourceforge.net/) or [NSubstitute](http://nsubstitute.github.io/) for creating your test doubles instead of hand-rolling them as is my preference. 

You may at this point be thinking there is a risk of inconsistency between the API when it is running against your Startup class and your InMemoryStartup, as these both need to be very similar but also different enough to allow for testing. There are approaches to reduce this risk to a minimum which I plan to cover in a follow-up post as this one has already gone on for long enough.

I hope this was useful for you and had enough depth to get you started with this style of in memory API testing. 