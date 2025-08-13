---
title: "HowTo: Use Decorator Pattern to extend services implementations"
author: "Dario Airoldi"
date: "2025-08-12"
categories: [news, code, copilot, development]
image: "image.jpg"
draft: true
---

# Introduction

**Decorator pattern** can be useful to **extend services implementation without modifying their code**.

This is expecially useful in codebases such as diginsight where **services are implemented across different assemblies**, depending on their target use and dependencies.

In Core assemblies, services often have minimal and most open implementation, while in other assemblies, they may have more specific logic related to their specific dependencies.

![core assemblies and platform specific assemblies](<images/001.01a core assemblies and platform specific assemblies.png>)

# More Information


