---
title: "HowTo: Manage Sensitive Configurations with Config Injection from Private Repositories"
author: "Dario Airoldi"
date: "2024-12-14"
categories: [news, code, development]
image: "image.jpg"
draft: false
---

## How to Manage Sensitive Configurations in Public Repositories

When developing applications in public repositories, handling sensitive configuration files (such as secrets, keys, or internal URLs) is a common challenge. Exposing these files can lead to security risks, while omitting them complicates development and deployment.

This guide presents a robust, practical solution: **store sensitive configurations in a private repository and inject them automatically during build and runtime**. This approach keeps your public codebase clean and secure, while ensuring a smooth developer experience.

---

## Table of Contents

1. [Why Is This a Problem?](#why-is-this-a-problem)
2. [Solution Overview](#solution-overview)
3. [Practical Example: Diginsight Approach](#practical-example-diginsight-approach)
4. [Step 1: Safe Local Development with External Configurations](#step-1-safe-local-development-with-external-configurations)
5. [Step 2: Automated Config Injection in CI/CD](#step-2-automated-config-injection-in-cicd)
6. [Benefits and Best Practices](#benefits-and-best-practices)
7. [Troubleshooting & FAQ](#troubleshooting--faq)
8. [References](#references)

---

## Why Is This a Problem?

- **Accidental Exposure:** Sensitive files can be mistakenly committed and leaked. Even if deleted, git history may retain them.
- **Lack of Versioning:** Excluding configs from git means lost change history and poor team coordination.
- **Complex Setup:** Developers must manually obtain and copy configs, leading to onboarding friction and errors.
- **Mix of Secret/Non-Secret Data:** Not all private data fits in a vault; some are just internal settings that shouldn’t be public.

---

## Solution Overview

**Keep sensitive configurations in a private repository** that mirrors your public repo’s structure.  
**Inject these files automatically** during local development and CI/CD builds, so no manual copying is needed and no secrets are ever committed to the public repo.

**Key points:**

- Public repo contains only example configs (no secrets).
- Private repo holds real configs, versioned and secured.
- Application and CI/CD pipeline are configured to load/merge configs from the private repo at runtime/build time.

---

## Practical Example: Diginsight Approach

For every public repository (e.g., [diginsight/components](https://github.com/diginsight/components)), there is a corresponding private repository (e.g., [diginsight/components.internal](https://github.com/diginsight/components.internal)) with the same folder structure, holding sensitive configs.

| Public Repository | Internal Repository |
|-------------------|--------------------|
| ![Public repo](<001.01 AuthenticationSampleAPI into the public repository.png>) | ![Private repo](<001.02 AuthenticationSampleAPI folder with configurations into the private repository.png>) |

- Public repo: only example configs, no secrets.
- Private repo: real configs, versioned and secure.

---

## Step 1: Safe Local Development with External Configurations

**Goal:** Allow developers to run the app locally with private configs, without copying files into the public repo.

**How:**

- Set an environment variable (e.g., `ExternalConfigurationFolder`) pointing to the private config folder.
- Application startup code loads configs from this external folder if available.

**Sample code:**

```csharp
// ...existing code...
var externalConfigurationFolder = Environment.GetEnvironmentVariable("ExternalConfigurationFolder");
if (isLocal && Directory.Exists(externalConfigurationFolder))
{
    // Load config from external folder
    // ...existing code...
}
// ...existing code...
```

**Result:**  
Developers just clone both repos, set the environment variable, and run the app—no manual copying required.

> Code for loading configurations from an '__ExternalConfigurationFolder__' folder is already available in the __Diginsight.Components__ component `WebHostBuilderExtensions.ConfigureAppConfiguration2` method.<br>
This code, or similar, can be easily integrated into any application startup sequence.<br>


![App running with private config](<002.01 - Diginsight_components_loading_config_from_external_folder.png>)

---

## Step 2: Automated Config Injection in CI/CD

**Goal:** During CI/CD builds (e.g., GitHub Actions), automatically inject private configs before building and deploying.

**How:**

1. Checkout both the public and private repos in the workflow.
2. Copy config files from the private repo to the appropriate locations in the public repo.
3. Proceed with build and deployment.

**Sample GitHub Actions steps:**

```yaml
- name: Checkout public repo
  uses: actions/checkout@v4

- name: Checkout private repo
  uses: actions/checkout@v4
  with:
    repository: diginsight/components.internal
    path: components.internal
    token: ${{ secrets.INTERNAL_REPOSITORY_TOKEN }}

- name: Copy configs
  run: |
    cp components.internal/src/Samples/AuthenticationSampleApi/appsettings.*.json src/Samples/AuthenticationSampleApi/
    # ...repeat for other configs...

- name: Build
  run: dotnet build src/Diginsight.Components.sln --configuration Release
```

**Result:**  
Builds and deployments always use the latest, secure configs—no secrets in the public repo, no manual steps.

---

## Benefits and Best Practices

- **Security by Design:** No secrets in public code, no risk of accidental leaks.
- **Versioning:** Private configs are tracked and auditable.
- **Developer Experience:** No manual copying; onboarding is easy.
- **Automation:** CI/CD pipelines handle config injection seamlessly.

**Best Practices:**

- Never commit real secrets to the public repo.
- Use environment variables or CI/CD secrets to access the private repo.
- Document the setup for new team members.
- Consider adding a simple diagram to illustrate the workflow.

---

## Troubleshooting & FAQ

**Q: What if the private repo is not accessible?**  
A: Only configuration files from the public repository are considered by the startup sequence.<br>
As with any public repository, the developer can adjust them replacing placeholders as reported by the documentation and run the application without the private repository.

**Q: What if configs are missing at runtime?**  
A: The app should log a clear error. Check that the environment variable or CI/CD copy step is correct.

**Q: How do I handle multiple environments (dev, prod, etc.)?**  
A: Store environment-specific configs in the private repo, and load the appropriate one based on environment variables.

---

## References

- [How to use private Git submodules](https://docs.readthedocs.io/en/stable/guides/private-submodules.html)
- [Using Private Git Submodules](https://me-readthedocs.readthedocs.io/en/latest/guides/private-submodules.html)

---

This approach ensures your sensitive configuration files are managed wisely, reducing risk and making life easier for developers and DevOps engineers alike.

---

