---
title: "HowTo: Manage Sensitive Configurations with Config Injection from Private Repositories"
author: "Dario Airoldi"
date: "2024-12-14"
categories: [news, code, development]
image: "image.jpg"
draft: false
---

# OVERVIEW
When developing applications in public, it’s common to use __configuration files__ that contain sensitive information.<br>
However, __managing these configurations in public repositories__ poses a challenge, as they should not be exposed.
<br>

This guide introduces a __robust solution to the problem__: __using a separate private repository for sensitive configuration files__ and __injecting those files during build time and at runtime__ to avoid the need of duplicating or manually copying them during build or development time.<br>

This article demonstrates how:

- '__injecting configurations from a private repository__' can be achieved with a few GitHub Actions steps.<br>
- '__loading configurations from an external folder__' can be achieved with a simple code change in the application startup sequence.<br>

Code for loading configurations from an '__ExternalConfigurationFolder__' folder is already available in the __Diginsight.Components__ component `WebHostBuilderExtensions.ConfigureAppConfiguration2` method.<br>
This code, or similar, can be easily integrated into any application startup sequence.<br>


# A PRACTICAL EXAMPLE

The approach proposed by this guide is used on our __diginsight repositories__.<br>
For every repository such as:<br>__[https://github.com/diginsight/components](https://github.com/diginsight/components)__<br>
a corresponding '__.internal__' (private) repository:<br>__[https://github.com/diginsight/components.internal](https://github.com/diginsight/components.internal)__<br> 
mirroring the folder structure of the public repository, is used to hold sensitive configurations.<br>

As an example, the __AuthenticationSampleApi__ exists in the public repository, under folder '__src/samples__'.<br>
Configurations for that sample are stored within the corresponding __AuthenticationSampleApi__ folder in the private repository (under '__src/samples__' folder, within the '__.internal__' repository).


| public repository  | internal repository | 
|-----------|-----------|
| ![alt text](<001.01 AuthenticationSampleAPI into the public repository.png>) | ![alt text](<001.02 AuthenticationSampleAPI folder with configurations into the private repository.png>) |


The '__diginsight/components__' public repo contains only example configurations, with no sensitive data.<br>
The real samples configuration files are stored into the '__diginsight/components.internal__' repository, with proper versioning and security.<br>

The following paragraphs explain how:

- __CI/CD GitHub Actions__ can easily inject the configuration files from the '__.internal__' repository before deploying code to the desired azure resources.<br>
- the __application startup sequence__ can be easily customized to allow merging configuration files from an '__ExternalConfigurationFolder__' environment variable.<br>
In this way, developers only need to clone the '__components__' and '__components.internal__' repositories, and the code will work immediately.

The image below shows the __AuthenticationSampleApi__ running with a private configuration '__Testms__' from the external folder '__..\..\..\..Diginsight\components.internal\src\Samples\AuthenticationSampleApi__'.<br>
![alt text](<002.01 - Diginsight_components_loading_config_from_external_folder.png>)

The developer just needs to open and run the sample, without any manual steps to copy configuration files from the private repository to the public repository.<br>


# ADDITIONAL DETAILS

Before diving into the solution, let’s recap why handling sensitive config in public repos can be a problem:

- __Accidental Exposure__: It’s frighteningly easy for someone to mistakenly commit a file with passwords or keys. Once pushed to a public repo, secrets can be immediately compromised. Even if removed later, history might still contain them.<br>
- __Lack of Versioning__: When we exclude configs from git, we lose the ability to track changes. Teams might struggle to keep everyone’s config in sync, and changes to secrets (e.g., rotating a key) aren’t documented in code.<br>
- __Complex Setup__: Developers need a reliable way to get the required config values to run the app. Without a good system, onboarding new contributors or deploying to new environments can be error-prone (passing around config files manually, etc.).<br>
- __Mix of Secret/Non-Secret Data__: Not all sensitive info is a “secret” that can go into a vault. Some are just private identifiers or settings (client IDs, internal URLs, personal data) that you don’t want public. These need protection too, but handling them differs from handling, say, passwords.<br> 

Keeping our sensitive configurations in a private repo ensures by design:

- __Isolation__: keep sensitive data out of the public repo by default,<br>
- __Version Control__: still track and manage those configs in a secure way,<br><br>
This approach works as long as we provide:<br>
- __Safety__: configs are automatically loaded at application startup, without requiring the developer to manually copy them to the public repository file system clone, that would bring the risk of accidental exposure.
- __Automation__: configs are automatically injected during builds developments with no manual steps.<br>


## Step 1 (Safety): merge config files from an external folder, at application startup

Applications code can be easily instructed during the startup sequence to load configurations from an external folder specified by an `externalConfigurationsFolder` variable. This allows the application to dynamically load configurations from a secure location outside the public repository.
This enables the application to load configurations from a secure location outside the public repository.

Example code from `ConfigureAppConfiguration2` in __Diginsight.Components__ component follows this approach:
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
For this reason, `AuthenticationSampleApi` can be run with an external configuration `Testms` from the external folder `components.internal\src\Samples\AuthenticationSampleApi`.
![alt text](<001.03 AuthenticationSampleApi running with private configuration Testms.png>)

The developer can run the sample without need of copying the configuration files from the private repository to the public repository.<br>


## Step 2 (Automation): Inject configuration files from an external repository, during GitHub actions build steps

At build time, the application (or CI pipeline) can pull in the config from the private repo and merge it with the application.<br> Essentially, the app loads its normal configuration from the public files, then overrides or supplements those settings with values from the private repository.<br> 

The following yml code shows the __GitHub actions__ steps to inject the configuration files from the private repository into the public repository during build time.<br>

### step1 : Checkout the private repo

The following yml code from diginsight/components samples shows the github action:
- checkout of the current repo and 
- checkout of the __.internal__ repo<br>

```yaml
  - name: Checkout Repository
    uses: actions/checkout@v4

  - name: Checkout 'components.internal' Repository
    uses: actions/checkout@v4
    with:
      repository: diginsight/components.internal
      path: components.internal
      token: ${{ secrets.INTERNAL_REPOSITORY_TOKEN }}
      # If the repo is private, you need a token with access:
```

### step2 : copy configuration files from the private repository to the public repository
After dotnet restore, before the build step, 
the following code copies the configuration files from the private repository to the public repository.<br>

```yaml
  - name: Restore Samples 
    run: dotnet restore --interactive src/Diginsight.Components.sln

  - name: Copy AuthenticationSampleApi appsettings.*.json from components.internal
    run: |
      cp components.internal/src/Samples/AuthenticationSampleApi/appsettings.*.json src/Samples/AuthenticationSampleApi/
  - name: Copy AuthenticationSampleServerApi appsettings.*.json from components.internal
    run: |
      cp components.internal/src/Samples/AuthenticationSampleServerApi/appsettings.*.json src/Samples/AuthenticationSampleServerApi/

  - name: Build Samples
    run: dotnet build src/Diginsight.Components.sln --configuration Release
```

At this point, the sample configuration files are merged with the Diginsight Components files, so the GitHub Action can continue with the build and publish steps as usual.

# CONCLUSION

Managing sensitive configurations via config injection from a private repository offers __ease of use__ and __security by design__.<br>

By separating sensitive config into its own version-controlled private repo and integrating it into your app’s workflow, you get the best of both worlds: 

- __public code remains clean__ and safe
- __private data is handled in a controlled, auditable manner__.
- __Versioning__ is achieved through the private config repo (no more lost change history on configs).

This guide explains how we can eliminate the pain points for this approach:

- __Development experience is ensured__: the application startup can be customized to load configuration files from the private repository.
This allows developers to run the application with the private configuration files without needing to copy configuration files with sensitive information from the private to the public repository.

Code to load configuration files from an external folder ('__ExternalConfigurationFolder__') is available in the __Diginsight.Components__ component, in the `WebHostBuilderExtensions.ConfigureAppConfiguration2` method.
This code, or similar, can be easily integrated into any application's startup sequence.

- __Automation is preserved__ as the pipeline can be customized to load configuration files from the private repository during the build process.

- __Accidental leaks are eliminated by design__ as the developer can use the public repository without needing to copy configuration files with sensitive information into it.

Setting up dual repositories and configuration file injection requires some initial effort, but once in place, it streamlines your workflow and fortifies your project’s security posture.

If you are developing an application with a public (or widely shared) codebase that needs to handle private configuration data, consider structuring it with a private configuration file repository and injection mechanism.

It will give you control over your sensitive information, traceability of changes, and confidence during deployments—all while keeping your public repository truly public and clean.

This “HowTo” method ensures that sensitive configuration files are managed wisely, reducing risk and making life easier for developers and DevOps engineers alike.



# REFERENCE
This article analyzes an easy solution for __managing private configurations for public code repositories__.<br>
By following these steps, you can ensure that sensitive information remains secure while maintaining the flexibility and accessibility of your public codebase.

Additional resources for further reading:

- [How to use private Git submodules](https://docs.readthedocs.io/en/stable/guides/private-submodules.html)<br>
- [Using Private Git Submodules](https://me-readthedocs.readthedocs.io/en/latest/guides/private-submodules.html)

