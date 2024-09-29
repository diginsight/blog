---
title: "Steps to add diginsight to SampleWebApi01"
author: "Dario Airoldi"
date: "2024-09-29"
categories: [news, code, analysis]
image: "image.jpg"
draft: false
---

Today I worked with some friends at __integrating diginsight to a new web API__.

In this post I propose the steps we followed applied to the sample 
`SampleWebApi01` available into the `telemetry_samples` repository.
<br>

![SampleWeb API project](<001.01a - SampleWebApi project.png>)

You can use the steps below to obtain __diginsight text based streams__ on __Log4Net__ and the appservice __Console__.

# STEP01: Add references to diginsight packages
I start adding the following 3 references:

``` xml
<PackageReference Include="Diginsight.AspnetCore" Version="3.0.0-alpha.205" />
<PackageReference Include="Diginsight.Diagnostics" Version="3.0.0-alpha.205" />
<PackageReference Include="Diginsight.Diagnostics.Log4Net" Version="3.0.0-alpha.205" />
``` 
where:<br>
- `Diginsight.Diagnostics` is the core engine for application flow rendering<br>
- `Diginsight.Diagnostics.Log4Net` integrates Log4Net file logs <br>
- `Diginsight.AspnetCore` allows support for Dynamic Logging and Dynamic Configuration.<br>

![alt text](<002.01d - Add references to diginsight packages.png>)

# STEP02: Add the Observability `ActivitySource` to your projects 
`System Diagnostics` emits telemetry by means of `ActivitySource` classes.

In our solution we'll choose to use one `ActivitySource` class for every project, defined as below:

``` c#
internal static class Observability
{
    public static readonly ActivitySource ActivitySource = new(Assembly.GetExecutingAssembly().GetName().Name!);
}
```

![alt text](<002.02a - Add ActivitySource classes.png>)


# STEP03: Configure the startup sequence with 'AddObservability()' method
At this point we can configure the `Log4Net` file stream logger and the `Console` logger within the service __startup sequence__.<br>

we use an `AddObservability` extension method:

``` c#
public static void Main(string[] args)
{
    var app = default(WebApplication);

        var builder = WebApplication.CreateBuilder(args); 
        builder.Host.ConfigureAppConfigurationNH(); 

        builder.Services.AddObservability(builder.Configuration);     // Diginsight: registers loggers

        //...
        //... register services 
        //...

        app = builder.Build(); logger.LogDebug("app = builder.Build();");

        logger.LogInformation("Configure the HTTP request pipeline.");
        if (app.Environment.IsDevelopment())
        {
            app.UseSwagger();
            app.UseSwaggerUI(c =>
            {
                c.SwaggerEndpoint("/swagger/v1/swagger.json", "KnowledgeAPI v1");
                c.OAuthClientId(azureAd.ClientId);
                c.OAuthUsePkce();
                c.OAuthScopeSeparator(" ");
            });
        }

        app.UseHttpsRedirection();

        app.UseAuthorization();

        app.MapControllers();
    }

    app.Run();
}
``` 

where `builder.Services.AddObservability(builder.Configuration);` registers loggers for `Log4net` and the application `Console` as shown below:<br>

```c#
public static class AddObservabilityExtension
{
    public static IServiceCollection AddObservability(this IServiceCollection services, IConfiguration configuration)
    {
        // registers http context accessor
        // this is used to manage dynamic logging and dynamic configuration
        // when calls land to the controllers methods
        services.AddHttpContextAccessor(); 
        services.TryAddSingleton<IActionContextAccessor, ActionContextAccessor>();

        // registers Logging providers for the Console and Log4Net
        services.AddLogging(
            loggingBuilder =>
            {
                loggingBuilder.AddConfiguration(configuration.GetSection("Logging"));
                loggingBuilder.ClearProviders();

                if (configuration.GetValue("AppSettings:ConsoleProviderEnabled", true))
                {
                    // registers Logging providers for the Console
                    loggingBuilder.AddDiginsightConsole(configuration.GetSection("Diginsight:Console").Bind);
                }

                if (configuration.GetValue("AppSettings:Log4NetProviderEnabled", false))
                {
                    // registers Logging providers for Log4Net
                    loggingBuilder.AddDiginsightLog4Net(
                        static sp =>
                        {
                            IHostEnvironment env = sp.GetRequiredService<IHostEnvironment>();
                            string fileBaseDir = env.IsDevelopment()
                                ? Environment.GetFolderPath(Environment.SpecialFolder.UserProfile, Environment.SpecialFolderOption.DoNotVerify)
                                : $"{Path.DirectorySeparatorChar}home";

                            return new IAppender[]
                            {
                                new RollingFileAppender()
                                {
                                    File = Path.Combine(fileBaseDir, "LogFiles", "Diginsight", typeof(Program).Namespace!),
                                    AppendToFile = true,
                                    StaticLogFileName = false,
                                    RollingStyle = RollingFileAppender.RollingMode.Composite,
                                    DatePattern = @".yyyyMMdd.\l\o\g",
                                    MaxSizeRollBackups = 1000,
                                    MaximumFileSize = "100MB",
                                    LockingModel = new FileAppender.MinimalLock(),
                                    Layout = new DiginsightLayout()
                                    {
                                        Pattern = "{Timestamp} {Category} {LogLevel} {TraceId} {Delta} {Duration} {Depth} {Indentation|-1} {Message}",
                                    },
                                },
                            };
                        },
                        static _ => Level.All
                    );
                }
            }
        );

        // Loads options for diginsight activities
        // this decides which activities should be traced into the text based streams 
        // for the console and Log4Net
        services.ConfigureClassAware<DiginsightActivitiesOptions>(configuration.GetSection("Diginsight:Activities"))
                .DynamicallyConfigureClassAwareFromHttpRequestHeaders<DiginsightActivitiesOptions>();

        // Register DefaultDynamicLogLevelInjector for support of Dynamic logging
        services.AddDynamicLogLevel<DefaultDynamicLogLevelInjector>();

        return services;
    }

}
```
Please, note that `DiginsightActivitiesOptions` are read from `Diginsight:Activities` the configuration section shown below:
![alt text](<003.00 DiginsightActivitiesOptions read.png>)

(Working code is available into `SampleWebApi01` sample, within `telemetry_samples` repository)




# STEP04 (Opt): add Instrumentation to the startup sequence
Startup sequence may be tricky and often hides startup issues that are very difficult to debug.

Diginsight __provides full observability of the startup sequence__ by means of the `DeferredLoggerFactory`.
<br><br>
In particular, the `DeferredLoggerFactory`:<br>
- __gathers the execution flow until the the standard logging system is setup__.<br>
- __Flushes recorded telemetry__ upon services creation, __as soon as `ILogger<>` providers are installed__.<br>

here we create the deferred logger factory at application startup:
``` c#
public static IDeferredLoggerFactory LoggerFactory;

static Program()
{
    DiginsightActivitiesOptions activitiesOptions = new() { LogActivities = true };
    IDeferredLoggerFactory deferredLoggerFactory = new DeferredLoggerFactory(activitiesOptions: activitiesOptions);
    deferredLoggerFactory.ActivitySourceFilter = (activitySource) => activitySource.Name.StartsWith($"DS.");
    LoggerFactory = deferredLoggerFactory;
}
``` 
deferredLoggerFactory implements in memory recording of the application flow, until ILogger services are created.

After ILogger services creation, `FlushOnCreateServiceProvider` can be used to register Flush of the recorded application flow and redirection of the deferredLoggerFactory to the registered LoggerFactory.

The real Log flush is provided at services creation time by means of the __Diginsight ServiceProvider__.
the __Diginsight ServiceProvider__ is registered by means of `UseDiginsightServiceProvider` method as shown below.

The snippet below shows the `main` method where the startup flow is made observable by means of `FlushOnCreateServiceProvider` and `UseDiginsightServiceProvider`.

```c#
public static void Main(string[] args)
{
    ILogger logger = LoggerFactory.CreateLogger(typeof(Program));

    var app = default(WebApplication);
    using (var activity = Observability.ActivitySource.StartMethodActivity(logger, new { args }))
    {
        var builder = WebApplication.CreateBuilder(args); logger.LogDebug("builder = WebApplication.CreateBuilder(args);");
        builder.Host.ConfigureAppConfigurationNH(); logger.LogDebug("builder.Host.ConfigureAppConfigurationNH();");
        builder.Services.AddObservability(builder.Configuration);     // Diginsight: registers loggers
        builder.Services.FlushOnCreateServiceProvider(LoggerFactory); // Diginsight: registers startup log flush

        // ... 
        // services registration omitted.
        // ... 

        var webHost = builder.Host.UseDiginsightServiceProvider(); // Diginsight: Flushes startup log and initializes standard log
        app = builder.Build(); logger.LogDebug("app = builder.Build();");

        logger.LogInformation("Configure the HTTP request pipeline.");
        if (app.Environment.IsDevelopment())
        {
            app.UseSwagger();
            app.UseSwaggerUI(c =>
            {
                c.SwaggerEndpoint("/swagger/v1/swagger.json", "KnowledgeAPI v1");
                c.OAuthClientId(azureAd.ClientId);
                c.OAuthUsePkce();
                c.OAuthScopeSeparator(" ");
            });
        }

        app.UseHttpsRedirection();

        app.UseAuthorization();


        app.MapControllers();

    }

    app.Run();
}
```

# STEP05: add Instrumentation log to methods and return values
We are now ready to add automatic instrumentation to methods and return values:

here is an example method start
```c#
[HttpGet(Name = "GetWeatherForecast")]
public IEnumerable<WeatherForecast> Get()
{
    using var activity = Observability.ActivitySource.StartMethodActivity(logger);

```

here is an example method completion where the result value is added to the method closing row:
```c#
    activity?.SetOutput(ret);
    return ret;
}
```

the real code may looks as shown below:
![alt text](<003.02 - MethodActivity for automatic flow gathering.png>)

here is the resulting flow for the service startup and an API call:
![alt text](<003.01a - resulting flow.png>)
