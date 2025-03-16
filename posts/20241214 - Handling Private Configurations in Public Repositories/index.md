---
title: "HowTo: Handle Private Configurations in Public Repositories"
author: "Dario Airoldi"
date: "2024-12-14"
categories: [news, code, deelopment]
image: "image.jpg"
draft: false
---

# OVERVIEW

When developing applications, it's common to use __configuration__ files that contain __sensitive information__. <br>
However, __managing these configurations in public repositories__ poses a challenge, as they should not be exposed to the public.<br>

This article addresses the problem of handling private configurations when testing code in public repositories and proposes a solution __using a private repository__ associated with the __original repository__.

For example, consider the public repository __https://github.com/diginsight/components__.<br> 
We can create a corresponding private repository, __https://github.com/diginsight/components.internal__, to store private configurations.<br>

In this scenario, the __AuthenticationSampleApi__ exists in the public repository with public example configurations.<br>
We can define a __AuthenticationSampleApi folder__ in the private repository, mirroring the structure of the public repository, and store the private configuration within the src folder of the private repository.

| public repository  | internal repository | 
|-----------|-----------|
| ![alt text](<001.01 AuthenticationSampleAPI into the public repository.png>) | ![alt text](<001.02 AuthenticationSampleAPI folder with configurations into the private repository.png>) |



# ADDITIONAL DETAILS
To load and use private configurations from the public repository, we can consider two steps:
- Step 1: Load Configurations from an External Folder
- Step 2 (optional): load configurations from __Git Submodules__

## Step 1: Load Configurations from an External Folder
In this step, the code is instructed during the startup sequence to load configurations from an external folder specified by an `externalConfigurationsFolder` variable. <br>
This allows the application to dynamically load configurations from a secure location outside the public repository.

Example code from `ConfigureAppConfiguration2` in __Diginsight.Components.Configuration__ component follows this approach:
-  an __'ExternalConfigurationFolder'__ variable is read 
- if existing, the __environment configuration file__ is loaded from that folder instead of the current folder.<br>

```c#
public static void ConfigureAppConfiguration2(IHostEnvironment environment, IConfigurationBuilder builder, ILoggerFactory loggerFactory, Func<IDictionary<string, string>, bool>? tagsMatch = null)
{

    bool isLocal = environment.IsDevelopment();
    var environmentName = environment.EnvironmentName;
    int appsettingsEnvironmentIndex = GetJsonFileIndex($"appsettings.{environmentName}.json", builder);

    var appsettingsFileName = $"appsettings.{environmentName}.json";
    var appsettingsFilePath = appsettingsFileName;

    // in case 'ExternalConfigurationFolder' variable exists, 
    // environment Json Configuration is added from the external folder 
    var externalConfigurationFolder = Environment.GetEnvironmentVariable("ExternalConfigurationFolder");
    var externalConfigurationFolderExists = externalConfigurationFolder is not null && Directory.Exists(externalConfigurationFolder);
    if (isLocal && externalConfigurationFolderExists && !File.Exists(appsettingsFilePath))
    {
        var externalConfigurationFolderDirectoryInfo = new DirectoryInfo(externalConfigurationFolder!);

        var potentialAppsettingsFolder = externalConfigurationFolderDirectoryInfo.FullName;
        var potentialFilePath = Path.Combine(potentialAppsettingsFolder, appsettingsFileName);
        if (File.Exists(potentialFilePath))
        {
            appsettingsFilePath = potentialFilePath;
        }

        AppendLocalJsonFile(appsettingsFilePath, appsettingsEnvironmentIndex, builder, isLocal);
        builder.Sources.RemoveAt(appsettingsEnvironmentIndex);
    }


    IConfiguration configuration = builder.Build();

    // after loading private configurations, 
    // Azure Key Vault and other resources can be accessed
    var kvUri = configuration["AzureKeyVault:Uri"];
    if (!string.IsNullOrEmpty(kvUri))
    {
        var clientId = configuration["AzureKeyVault:ClientId"];
        var tenantId = configuration["AzureKeyVault:TenantId"];
        var clientSecret = configuration["AzureKeyVault:ClientSecret"];
        var applicationCredentialProvider = new ApplicationCredentialProvider(environment);

        var credential = applicationCredentialProvider.Get(tenantId, clientId, clientSecret);
        builder.AddAzureKeyVault(new Uri(kvUri), credential, new KeyVaultSecretManager2(DateTimeOffset.UtcNow, tagsMatch));
    }

    ...
}
```

The code snippet below shows `AuthenticationSampleApi` startup sequence using `ConfigureAppConfiguration2` to load configurations.<br>

``` c#
public static void Main(string[] args)
{
    var activitiesOptions = new DiginsightActivitiesOptions() { LogActivities = true };
    DeferredLoggerFactory = new DeferredLoggerFactory(activitiesOptions: activitiesOptions);
    DeferredLoggerFactory.ActivitySourceFilter = (activitySource) => true; 
    var logger = DeferredLoggerFactory.CreateLogger<Program>();

    IWebHost host;
    using (var activity = Observability.ActivitySource.StartMethodActivity(logger, new { args }))
    {
        host = WebHost.CreateDefaultBuilder(args)
            .ConfigureAppConfiguration2(DeferredLoggerFactory)
            .UseStartup<Startup>()
            .ConfigureServices(services =>
            {
                var logger = DeferredLoggerFactory.CreateLogger<Startup>();
                using var innerActivity = Observability.ActivitySource.StartRichActivity(logger, "ConfigureServicesCallback", new { services });

                services.TryAddSingleton(DeferredLoggerFactory);
            })
            .UseDiginsightServiceProvider()
            .Build();

        logger.LogDebug("Host built");
    }

    host.Run();
}
```
For this reason, `AuthenticationSampleApi` can be run with an external configuration `Testms` from the external folder `E:\dev.darioa.live\Diginsight\components.internal\src\Samples\AuthenticationSampleApi`.
![alt text](<001.03 AuthenticationSampleApi running with private configuration Testms.png>)


## (Optional) Step 2: Use Git Submodules
In this step, the __private repository folders are mapped as submodules of the public repository__. <br>
This allows the public repository to load configurations from a custom folder within it, ensuring that sensitive configurations are kept secure in the private repository.

Steps to set up Git submodules:

- Add the private repository as a submodule:
  ```
  git submodule add git@github.com:yourusername/components.internal.git config
  ```
- Update the .gitmodules file:
  ```
  [submodule "config"]
    path = config
    url = git@github.com:yourusername/components.internal.git
  ```
- Initialize and update submodules when cloning the repository:
  ```
  git clone --recurse-submodules git@github.com:yourusername/components.git
  cd components
  git submodule update --init --recursive
  ```
- Access the private configuration in your code:
  configurations from the submodule can now be used just setting the
  `externalConfigurationsFolder` variable to the submodule folder.


# REFERENCE
This article analyzes an easy solution for __managing private configurations for public code repositories__.<br>
By following these steps, you can ensure that sensitive information remains secure while maintaining the flexibility and accessibility of your public codebase.

Additional resources for further reading:
- [How to use private Git submodules](https://docs.readthedocs.io/en/stable/guides/private-submodules.html)<br>
- [Using Private Git Submodules](https://me-readthedocs.readthedocs.io/en/latest/guides/private-submodules.html)

