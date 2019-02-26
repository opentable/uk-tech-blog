---
layout: post
title: Structuring large ASP.NET Core projects for testing and readability
date: 2019-02-26 12:00:00
author: dgiddins
tags: [Testing, DotNetCore, ASP.Net, NUnit]
---

## TL;DR

This is a continuation of the post [Testing an API In Memory in ASP.NET Core](/blog/2019/01/28/testing-an-api-in-memory-in-asp-net-core/) where I described in detail how to test an ASP.NET Core API end-to-end in an isolated, repeatable fashion. This is a shorter post that discusses an approach to structuring your project that should make it easier to test and develop.

Oh and there are lots of code samples again!

## Where we left off
In my previous post I described how to get a truly isolated in-memory instance of a ASP.NET Core API configured in a test harness to perform repeatable tests. We created an InMemoryStartup class (for configuring the site for testing) and an InMemoryApi class (for encapsulating Test Doubles and setting up the API). I did not go into detail about the content of either of the Start up classes which is what I intend to address here.

Much of the complexity from this kind of testing derives from having to maintain two configuration classes and keeping them in sync correctly. It is all too easy for strange bugs to creep in and go unnoticed when differences are not correctly replicated. 

The solution to this is actually quite straightforward which is to create shared configuration classes that can be specialised for your in-memory testing situation. My solution involves creating two configuration classes, one for each of the two methods that are implemented by convention in your regular Startup.cs and the two methods of the IStartup interface. These are:

* WebAppConfigurator - for the Configure method of Startup
* ServiceCollectionInstallerRunner - for the ConfigureServices method of Startup

Both of these classes will have in-memory versions of themselves implemented as extension classes - we will come on to this later. 

## WebAppConfigurator
I know, not the best name but it describes what we are doing here and it's hard to name configuration and boot strapping classes properly. This class will look roughly as follows

```csharp

    public class WebAppConfigurator
    {
        private IConfiguration Configuration { get; set; }

        public WebAppConfigurator(IConfiguration configuration)
        {
            Configuration = configuration;
        }

        public void Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory, IApplicationLifetime appLifeTime, IServiceProvider serviceProvider)
        {
            //This is all just an example of the configuration steps you might need.
            ConfigureLogger(app, loggerFactory, serviceProvider); 
            ConfigureSwagger(app);

            ConfigureShutdown(app, appLifeTime);
            ConfigurePostStartup(app, appLifeTime);

            LocaleConfiguration.ConfigureRequestLocalization(app);
            app.UseResponseCompression();
            app.UseMvc();
        }

        protected virtual void ConfigureLogger((IApplicationBuilder app, ILoggerFactory loggerFactory, IServiceProvider serviceProvider) 
        {
            //configure the logger
        }
        
        protected virtual void ConfigureSwagger((IApplicationBuilder app) 
        {
            //configure swagger
        }
    }

```

This would be called from your startup class as follows

```csharp

    public class Startup
    {
        private IConfiguration Configuration { get; set; }

        public Startup(IHostingEnvironment env)
        {
            BuildConfiguration();
        }

        public void Configure(IApplicationBuilder applicationBuilder, IHostingEnvironment hostingEnvironment, ILoggerFactory loggerFactory, IApplicationLifetime appLifeTime, IServiceProvider serviceProvider)
        {
            var configurator = new WebAppConfigurator(Configuration);
            configurator.Configure(applicationBuilder, hostingEnvironment, loggerFactory, appLifeTime, serviceProvider);
        }

        //more to follow...

        private void BuildConfiguration()
        {
            var builder = new ConfigurationBuilder()
                .AddEnvironmentVariables();

            Configuration = builder.Build();
        }
    }

```

Notice that compared to the Startup class in my first post, the Configure method has additional parameters. The number of parameters you define in your Startup.Configure() method is flexible depending on what you require for your configuration. In my particular case I required all of the listed parameters. This caused some difficulties when implementing the IStartup interface whose Configure method only takes an IApplicationBuilder.

The next step is to create an in-memory version of the WebAppConfigurator which derives from the WebAppConfigurator. It will then override implementations of any virtual methods in this class which we use to encapsulate configuration steps.

This would look as follows:

```csharp

    public class InMemoryWebAppConfigurator : WebAppConfigurator
    {
        public InMemoryWebAppConfigurator(IConfiguration configuration) : base(configuration) { }

        protected override void ConfigureSwagger(IApplicationBuilder app)
        {
        }

        protected override void ConfigureLogger(IApplicationBuilder app, ILoggerFactory loggerFactory, IServiceProvider serviceProvider)
        {
        }
    }

```

In this example we don't really want swagger running in the test harness nor do we want logging (which in our case is writing to Redis) and so the configuration methods do nothing. The configuration steps that are encapsulated with this method are entirely up too you and depends on your particular use case.

The final step is to now call this from our InMemoryStartup

```csharp

    public class InMemoryStartup : IStartup
    {
        private IConfiguration Configuration { get; set; }

        public InMemoryStartup()
        {
            BuildConfiguration();
        }

        public IServiceProvider ConfigureServices(IServiceCollection services)
        {
            //todo
        }

        public void Configure(IApplicationBuilder app)
        {
            var configurator = new InMemoryWebAppConfigurator(Configuration);
            var nullLoggerFactory = new NullLoggerFactory();
            configurator.Configure(
                app,
                new HostingEnvironment()
                {
                    ApplicationName = "Test Harness",
                    EnvironmentName = "Test"
                },
                nullLoggerFactory,
                new ApplicationLifetime(new Logger<ApplicationLifetime>(new NullLoggerFactory())), null);
        }

        private void BuildConfiguration()
        {
            var builder = new ConfigurationBuilder()
                .AddEnvironmentVariables();
            Configuration = builder.Build();
        }
    }

```

Notice in the Configure method that I have constructed a HostingEnvironment type and provided some dummy data as well as passing in a NullLoggerFactory and an ApplicationLifetime type. In my case I did not need to pass the IServiceProvider so I passed null, but you can pass it after it is constructed in your ConfigureServices method.

## ServiceCollectionInstallerRunner
The next step is to configure all of your dependencies. In ASP.NET WebApi I tended to leverage 3rd party dependency injection containers such as Castle Windsor, but in ASP.NET Core I find the built-in resolver works perfectly well; at least with a few additions. 

The first of these I would recommend you use is [Scrutor](https://github.com/khellang/Scrutor). This provides two extension methods to the IServiceCollection type; Scan and Decorate. Scan allows you to use convention-based registration, meaning you don't have to register each type individually. Decorate allows for type decoration. 

The second addition is actually something that you can easily add yourself by copying the code below.

```csharp

    public interface IServiceCollectionInstaller
    {
        void ConfigureService(IServiceCollection services, IConfiguration configuration);
    }

    public class ServiceCollectionInstallerRunner
    {
        public void ConfigureServices(IServiceCollection services, IConfiguration configuration)
        {
            var installers = GetServiceInstallers();

            foreach (IServiceCollectionInstaller installer in installers)
            {
                installer.ConfigureService(services, configuration);
            }
        }
        protected virtual IEnumerable<IServiceCollectionInstaller> GetServiceInstallers()
        {
            Assembly assembly = Assembly.GetAssembly(typeof(ServiceCollectionInstallerRunner));

            IEnumerable<Type> installerTypes = from type in assembly.GetTypes()
                where typeof(IServiceCollectionInstaller).IsAssignableFrom(type) && type != typeof(IServiceCollectionInstaller)
                select type;

            foreach (Type installerType in installerTypes)
            {
                var constructors = installerType.GetConstructors();
                if (constructors.Any(constructor => constructor.GetParameters().Length > 0))
                {
                    throw new NoParameterlessConstructorException(installerType);
                }
            }

            IEnumerable<IServiceCollectionInstaller> installers = installerTypes.Select(type => (IServiceCollectionInstaller) Activator.CreateInstance(type));
            return installers;
        }

        public class NoParameterlessConstructorException : Exception
        {
            public NoParameterlessConstructorException(Type installerType)
                : base($"Constructor containing parameters not allowed in IServiceCollectionInstaller. Unable to create type {installerType.Name}")
            {
            }
        }
    }

```

What this provides is very similar to Castle Windsor's IWindsorInstaller interface which lets you define type registration classes for different areas of your application. 

The way I prefer to use this is to divide my API project into areas such as logging, monitoring and application-specific areas, and then in each folder have an implementation of IServiceCollectionInstaller that will take care of configuring all the types in that folder. This keeps the configuration as close as possible to the parts being configured. For example:

```csharp
 
    public class MasstransitServiceInstaller : IServiceCollectionInstaller
    {
        public void ConfigureService(IServiceCollection services, IConfiguration configuration)
        {
            services.AddSingleton<IBus>(CreateBus);
            services.AddSingleton<IEventSender, EventSender>();
            services.Decorate<IEventSender, EventSenderFailureHandler>();
        }

        public IBus CreateBus(IServiceProvider provider)
        {
            var configuration = provider.GetService<IRabbitMqConfiguration>();

            IBusControl busControl = Bus.Factory.CreateUsingRabbitMq(sbc =>
            {
                var hostAddress = new Uri(configuration.RabbitMqUrl);
                var host = sbc.Host(hostAddress, h =>
                {
                    h.Username("guest");
                    h.Password("guest");
                });
                sbc.UseJsonSerializer();
                sbc.ReceiveEndpoint(host, "HA-OT.api-messages", c =>
                {
                    c.PrefetchCount = 50;
                    c.SetQueueArgument("ha", 1);
                    c.SetQueueArgument("tx", 1);
                });
            });
            return busControl;
        }
    }

```

In this example we are setting up MassTransit with RabbitMQ along with our own abstractions on top of the IBus and a decorator for that type to handle send failures.

This makes dependency configuration easy to follow and locate, and is quite rational instead of one giant configuration vomiting dependency configuration code over hundreds of lines in your Startup.cs file.

We can then complete our Startup class as follows

```csharp

    public class Startup
    {
        private IConfiguration Configuration { get; set; }

        public Startup(IHostingEnvironment env)
        {
            BuildConfiguration();
        }

        public void Configure(IApplicationBuilder applicationBuilder, IHostingEnvironment hostingEnvironment, ILoggerFactory loggerFactory, IApplicationLifetime appLifeTime, IServiceProvider serviceProvider)
        {
            var configurator = new WebAppConfigurator(Configuration);
            configurator.Configure(applicationBuilder, hostingEnvironment, loggerFactory, appLifeTime, serviceProvider);
        }

        public void ConfigureServices(IServiceCollection services)
        {
            var installerRunner = new ServiceCollectionInstallerRunner();
            installerRunner.ConfigureServices(services, Configuration);
        }

        private void BuildConfiguration()
        {
            var builder = new ConfigurationBuilder()
                .AddEnvironmentVariables();

            Configuration = builder.Build();
        }
    }

```

The next step is to make it possible to override the services as configured for running normally to work for our in-memory tests. For this we need to create an in-memory version of the ServiceCollectionInstallerRunner and a new interface that defines overrides of the Service Collection Installer. The code is shared below.

```csharp
    
    public interface IServiceCollectionInstallerOverride : IServiceCollectionInstaller
    {
        Type InstallerOverridden { get; }
    }

    public class InMemoryServiceCollectionRunner : ServiceCollectionInstallerRunner
    {
        private readonly List<IServiceCollectionInstallerOverride> _serviceCollectionInstallerOverrides;

        public InMemoryServiceCollectionRunner(List<IServiceCollectionInstallerOverride> serviceCollectionInstallerOverrides)
        {
            _serviceCollectionInstallerOverrides = serviceCollectionInstallerOverrides;
        }

        protected override IEnumerable<IServiceCollectionInstaller> GetServiceInstallers()
        {
            IEnumerable<IServiceCollectionInstaller> defaultList = base.GetServiceInstallers();

            var listWithOverrides = new List<IServiceCollectionInstaller>();

            foreach (IServiceCollectionInstaller defaultInstaller in defaultList)
            {
                var installerOverride =
                    _serviceCollectionInstallerOverrides.SingleOrDefault(override => override.InstallerOverridden == defaultInstaller.GetType());

                if (installerOverride != null)
                {
                    listWithOverrides.Add(installerOverride);
                }
                else
                {
                    listWithOverrides.Add(defaultInstaller);
                }
            }
            
            return listWithOverrides;
        }
    }

```

The way this works is that you create overrides for the service installers that install dependencies that you want to replace with a Test Double. Taking our previous example of the MasstransitServiceInstaller we can create the following:

```csharp

    public class MasstransitServiceInstallerOverride : IServiceCollectionInstallerOverride
    {
        private readonly FakeEventSender _fakeEventSender;

        public MasstransitServiceInstallerOverride(FakeEventSender fakeEventSender)
        {
            _fakeEventSender = fakeEventSender;
        }

        public void ConfigureService(IServiceCollection services, IConfiguration configuration)
        {
            services.AddSingleton<IEventSender>(_fakeEventSender);
        }

        public Type InstallerOverridden => typeof(MasstransitServiceInstaller);
    }

```

This would be used as follows

```csharp

    var serviceCollectionInstallerOverrides = new List<IServiceCollectionInstallerOverride>()
        {
            new MasstransitServiceInstallerOverride(_fakeEventSender),
        };

    var serviceCollectionRunner = new InMemoryServiceCollectionRunner(serviceCollectionInstallerOverrides);

```

We create a list of the overrides, and the overrides know which installer they will replace. The InMemoryServiceCollectionRunner then matches up overrides with installers and then runs the override to configure dependencies instead of the normal installer. 

This leads us to a version of the InMemoryApi that looks as follows

```csharp

    public class InMemoryApi
    {
        public InMemoryApi()
        {
            DoubledEventSender = new FakeEventSender();

            var serviceCollectionInstallerOverrides = new List<IServiceCollectionInstallerOverride>
                {
                    new MasstransitServiceInstallerOverride(DoubledEventSender)
                };

            var startUp = new InMemoryStartup(new InMemoryServiceCollectionRunner(serviceCollectionInstallerOverrides));
            var webHostBuilder = new WebHostBuilder().UseStartupInstance(startUp);
            var server = new TestServer(webHostBuilder);
            Client = server.CreateClient();
        }

        public IEventSender DoubledEventSender {get;private set;}
        public HttpClient Client {get; private set; }
    }

```

This might seem like a lot of complication to achieve more or less what we did in the previous post, but this approach is in my opinion much easier to scale to a large/complex project. When you have possibly 100 dependencies to manage and need to change some of them for in-memory testing this technique makes things easier to follow.

I would be the first to admit that this is an opinionated approach to the problem of structuring your ASP.NET Core projects. If this works for your particular situation then I am glad to have helped.