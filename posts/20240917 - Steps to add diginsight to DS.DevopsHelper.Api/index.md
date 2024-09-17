---
title: "Steps to add diginsight to DS.DevopsHelper.Api"
author: "Dario Airoldi"
date: "2024-09-17"
categories: [news, code, analysis]
image: "image.jpg"
draft: true
---

today I worked with some friends to a .net API and I wanted to have it instrumented with diginsight.

![DevopsHelper API project](<001.01 - DevopsHelper API project.png>)

below the basic steps I followed to integrate text based streams on Log4Net and the appservice Console.

# STEP01: Add references to diginsight packages
I start adding the following 3 references:

``` xml
<PackageReference Include="Diginsight.AspnetCore" Version="3.0.0-alpha.203" />
<PackageReference Include="Diginsight.Diagnostics" Version="3.0.0-alpha.203" />
<PackageReference Include="Diginsight.Diagnostics.Log4Net" Version="3.0.0-alpha.203" />
``` 
where:
- `Diginsight.Diagnostics` is the core engine for application flow rendering
- `Diginsight.Diagnostics.Log4Net` integrates Log4Net file logs 
- `Diginsight.AspnetCore` allows support for Dynamic Logging and Dynamic Configuration.

![alt text](<002.01a - Add references to diginsight packages.png>)

# STEP02: Add the Observability `ActivitySource` to your projects 
System Diagnostics emits telemetry by means of `ActivitySource` classes.

In our solution we'll choose to use one `ActivitySource` class for every project:

``` c#
internal static class Observability
{
    public static readonly ActivitySource ActivitySource = new(Assembly.GetExecutingAssembly().GetName().Name!);
}
```

![alt text](<002.02 - Add ActivitySource classes.png>)


# STEP03: Configure the startup sequence
Here we'll configure the `Log4Net` file stream logger and the `Console` logger.
Also, we'll add logging to the startup sequence itself by means of the `DeferredLoggerFactory`.
<br>
<br>
In particular, the `DeferredLoggerFactory`:<br>
- gathers the execution flow until the the standard logging system is setup.<br>
- Flushes recorded telemetry to standard `ILogger<>` system, as soon as possible, upon services creation.<br>

here we create the deferred logger factory at application startup:
``` c#
public static IDeferredLoggerFactory LoggerFactory;

static Program()
{
    DiginsightActivitiesOptions activitiesOptions = new() { LogActivities = true };
    IDeferredLoggerFactory deferredLoggerFactory = new DeferredLoggerFactory(activitiesOptions: activitiesOptions);
    deferredLoggerFactory.ActivitySources.Add(Observability.ActivitySource);
    LoggerFactory = deferredLoggerFactory;
}
``` 


here we use the deferred logger factory to create Ilogger for for the startup sequence:
``` c#
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

        //...
        //... register services 
        //...

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
please note that:
- `builder.Services.AddObservability(builder.Configuration);` registers loggers for Log4net and the application Console<br>
- `builder.Services.FlushOnCreateServiceProvider(LoggerFactory);` registers startup log flush that will happen upon builder.Build()<br>
- `var webHost = builder.Host.UseDiginsightServiceProvider();` registers a service provider that manages deferred log flush with services creation.<br>


# STEP04: add Instrumentation log to methods and return values
we are now ready to add automatic instrumentation to methods and return values:

here is an example method start
```c#
[HttpGet("SendApprovalNotifications")]
public async Task<IActionResult> SendApprovalNotificationsAsync(int buildId)
{
    using var activity = Observability.ActivitySource.StartMethodActivity(logger, new { buildId });
```

here is an example method completion where the result value is added to the method closing row:
```c#
    activity?.SetOutput(ret);
    return ret;
}
```

![alt text](<003.01 - MethodActivity for automatic flow gathering.png>)

here is the resulting flow for an API call:
![alt text](<003.01 - resulting flow.png>)

