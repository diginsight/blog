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

For example, consider the public repository __https://github.com/diginsight/tools__.<br> 
We can create a corresponding private repository, __https://github.com/diginsight/toolsinternal__, to store private configurations.<br>

In this scenario, the __MIPDocumentInspector tool__ exists in the public repository with public example configurations.<br>
We can define a __MIPDocumentInspector folder__ in the private repository, mirroring the structure of the public repository, and store the private configuration within the src folder of the private repository.

| Tools repository  | Toolsinternal repository | 
|-----------|-----------|
| ![alt text](<001.01 MipDocumentInspector public repository.png>)       | ![alt text](<002.01 MipDocumentInspector private configurations.png>) |



# ADDITIONAL DETAILS
To load and use private configurations from the public repository, we can consider two options:
- Option 1: Load Configurations from an External Folder
- Option 2: Use Git Submodules

## Option 1: Load Configurations from an External Folder
In this approach, the code is instructed during the startup sequence to load configurations from an external folder specified by an externalConfigurationsFolder variable. This allows the application to dynamically load configurations from a secure location outside the public repository.

Example code snippet:
```c#
public class Startup
{
    public IConfiguration Configuration { get; }

    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public void ConfigureServices(IServiceCollection services)
    {
        var externalConfigurationsFolder = Environment.GetEnvironmentVariable("EXTERNAL_CONFIGURATIONS_FOLDER");
        if (!string.IsNullOrEmpty(externalConfigurationsFolder))
        {
            var builder = new ConfigurationBuilder()
                .SetBasePath(externalConfigurationsFolder)
                .AddJsonFile("appsettings.testabb.json", optional: true, reloadOnChange: true);

            Configuration = builder.Build();
        }

        services.AddSingleton(Configuration);
        // Other service configurations
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        // Application configuration
    }
}
```

## Option 2: Use Git Submodules
In this approach, the __private repository folders are mapped as submodules of the public repository__. <br>
This allows the public repository to load configurations from a custom folder within it, ensuring that sensitive configurations are kept secure in the private repository.

Steps to set up Git submodules:

- Add the private repository as a submodule:
  ```
  git submodule add git@github.com:yourusername/toolsinternal.git config
  ```
- Update the .gitmodules file:
  ```
  [submodule "config"]
    path = config
    url = git@github.com:yourusername/toolsinternal.git
  ```
- Initialize and update submodules when cloning the repository:
  ```
  git clone --recurse-submodules git@github.com:yourusername/tools.git
  cd tools
  git submodule update --init --recursive
  ```
- Access the private configuration in your code:
  ``` c#
  public class Startup
  {
      public IConfiguration Configuration { get; }
  
      public Startup(IConfiguration configuration)
      {
          Configuration = configuration;
      }
  
      public void ConfigureServices(IServiceCollection services)
      {
          var builder = new ConfigurationBuilder()
              .SetBasePath("config/MIPDocumentInspector/src")
              .AddJsonFile("appsettings.testabb.json", optional: true,     reloadOnChange: true);
  
          Configuration = builder.Build();
  
          services.AddSingleton(Configuration);
          // Other service configurations
      }
  
      public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
      {
          // Application configuration
      }
  }
  ```

# REFERENCE
This article provides a easy solution for __managing private configurations in public repositories__ by using a private repository and Git submodules.<br>
By following these steps, you can ensure that sensitive information remains secure while maintaining the flexibility and accessibility of your public codebase.

- [How to use private Git submodules](https://docs.readthedocs.io/en/stable/guides/private-submodules.html)<br>
- [Using Private Git Submodules](https://me-readthedocs.readthedocs.io/en/latest/guides/private-submodules.html)

