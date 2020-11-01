---
title: "Upgrading to .NET 5.0"
excerpt_separator: "<!--more-->"
categories:
  - Post Formats
tags:
  - Post Formats
  - readability
  - standard
last_modified_at: 2020-10-25T22:47:00+01:00
---

# Upgrading to .NET 5.0

When we started building _Yet Another Cloud Accounting_ in October 2019 (a little over one year ago), .NET Core 3.0 has been out barely a month. Wanting to work with the latest and greatest, but needing a solid foundation that would support the stack we've envisioned, we went with .NET Core 2.2. There were two main reasons at the time. One, early on, .NET Core 3.0 had a reputation of having its share of bugs, enough so that we thought it better to wait until an LTS release with .NET Core 3.1. 
 Second, we wanted to use the most recent [OData v4](https://docs.microsoft.com/en-us/odata/) libraries and ASP.NET Core WebAPI, which at the time meant [Microsoft.AspNetCore.OData 7.2.2](https://www.nuget.org/packages/Microsoft.AspNetCore.OData/7.2.2) and [Microsoft.OData.Core 7.6.1](https://www.nuget.org/packages/Microsoft.OData.Core/7.6.1), both targeting .NET Standard 2.0. 

Later that year, in early December, .NET Core 3.1 was released and with it, we got the LTS release I felt was stable enough to upgrade to. Over Christmas, I upgraded our solution to a combination of .NET Standard 2.1 and .NET Core 3.1 projects with the latest OData libraries. A few breaking API changes popped up but nothing too difficult to fix. I made the changes in isolation on a new branch forked from our master (_this is the way_) and built it in its own Azure DevOps pipeline to observe for a few days how stable our new stack would prove to be. With no issues whatsoever, I merged the changes later in January when Microsoft.AspNetCore.OData package came out with support for .NET Core 3.1.

Ever since I’ve been keeping an eye on the next version of .NET Core. In May, the [official word](https://devblogs.microsoft.com/dotnet/introducing-net-5/) went out that this would be nothing less than .NET 5, a critical milestone unifying .NET Framework, .NET Core, Xamarin, and Mono. With one framework to rule them all, it looked a little daunting at first, but after reading up a bit on the master plan, ASP.NET runtime turned out to be the least affected. For what it’s worth, for ASP.NET, .NET 5 might as well be just another version number increment. Well, that makes it easier for me :) 

Nevertheless, I was looking forward to .NET 5 for the [new goodies in Entity Framework Core](https://docs.microsoft.com/en-us/ef/core/what-is-new/ef-core-5.0/plan). We lean on EF Core heavily for pretty much all data access but reporting. I frequent EF Core GitHub repo and the EF Core team has always been very transparent and open about what they’re cooking for the next release. Features like skip navigations, TPT (table-per-type) inheritance, or filtered includes are something I was anxious to try in our solution too.

I got to poke around .NET 5 first time around mid-March during the first (spring) Coronavirus lockdown when the [first preview](https://devblogs.microsoft.com/dotnet/announcing-net-5-0-preview-1/) bits were released. Baby steps, just a simple test of whether the new SDK would build our solution without any changes while still targeting .NET Core 3.1 runtime. Zero build errors, all tests green, nice. For a while, that was as far as I got. I progressively moved on to later preview versions as they came out and kept on building our solution with the latest SDK. The fact that I was able to use the latest .NET 5 SDK for months on my machine while the rest of the team and our CI was still on .NET 3.1 SDK looked like a good sign to take it further.

In the doom and gloom of the second (fall) Coronavirus lockdown came the announcement of the [first .NET 5 release candidate](https://devblogs.microsoft.com/dotnet/announcing-net-5-0-rc-1/). I downloaded it immediately and this time decided to target the new runtime as well, including upgrading all packages. As usual, I created a new branch and a new pipeline to test the impact before wider release to my dev team. After all, it is still not a final release and subject to change.

## New target framework

A major change in .NET 5 is the [departure from the .NET Standard](https://docs.microsoft.com/en-us/dotnet/standard/net-standard#net-5-and-net-standard) concept. There is no longer a distinction between a .NET specification and implementation. .NET 5 aims to be the one unified runtime that will support any .NET workload, be it web (ASP.NET), desktop (WinForms, WPF, UWP), or mobile (Xamarin). Going forward .NET Standard will not be updated as there won’t be multiple implementations to standardize. .NET Standard will still find its use with libraries that can be used by multiple .NET implementations. Although, even here we’re already seeing a gradual sunset of .NET Framework with the last applicable .NET Standard being version 2.0.

With one unified .NET implementation, the TFM will also be simplified and we will now only need _one_ TFM to specify a .NET runtime version. As a refresher, TFM stands for [target framework moniker](https://docs.microsoft.com/en-us/dotnet/standard/frameworks), a standardized token format for specifying the target framework of a .NET app or library. For .NET 5, this will be `net5.0`.

In our solution, this meant changing the target framework from .NET Core 3.1 and .NET Standard 2.1 (yes, we’ve had a few of those too) in every .csproj file to .NET 5 like so

```xml
<TargetFramework>net5.0</TargetFramework>
```

The few .NET Standard projects that we had were true libraries - collections of reusable code that was not specific to our solution but if there ever was the will, could have been packaged using Nuget and distributed. By targeting .NET Standard, the libraries could be used by anyone else irrespective of .NET runtime as long as it conformed to particular .NET Standard specification. Basically, these were our _commons_. With the external reuse unlikely and nowhere on our list of priorities, I also switched the libraries over to .NET 5. 

There is one other interesting development in .NET 5 with regards to target frameworks and that is the [introduction of OS-specific TFMs](https://docs.microsoft.com/en-us/dotnet/standard/frameworks#net-5-os-specific-tfms) in addition to the base TFM. Developers can now make use of platform-specific features at the expense of interoperability and target OS-specific runtimes instead of base runtime. When developing mobile apps for Android one can target `net5.0-android` and similarly `net5.0-tvos` for Apple TV apps. The OS-specific TFM will also allow the developer to specify an exact OS version such as `net5.0-ios12.0`.

## Breaking changes

As with any major release, we can expect to see breaking API changes. Although it's rarely something to look forward to and often leads to developers experiencing the five stages of grief, it's the price we must pay for keeping up with the rapid pace of innovation. This time around, there weren’t as many as with the upgrade to .NET Core 3.1.

### IStringLocalizer - removal of obsolete WithCulture() method

We use [Microsoft.Extensions.Localization](https://www.nuget.org/packages/Microsoft.Extensions.Localization/5.0.0-rc.1.20451.17) package to localize our UI labels and messages. [IStringLocalizer](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.localization.istringlocalizer) interface has been warning us for a while now that `WithCulture()` method is obsolete and will be removed in the future. No buts. [WithCulture()](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.localization.istringlocalizer.withculture?view=dotnet-plat-ext-3.1) is used to create culture-specific instances of `IStringLocalizer` by passing a `CultureInfo` object. The fix is simple as the obsolescence warning message itself suggests - change the `CultureInfo.CurrentCulture`, the culture for the current thread, to your desired culture (and don't forget to change it back later when done). As we were expecting this and had tests in place, I only needed to remove all remaining uses of `WithCulture()` including tests.

### INavigationBase - does not have IsOnDependent or ForeignKey properties

This one comes from [Microsoft.EntityFrameworkCore](https://www.nuget.org/packages/Microsoft.EntityFrameworkCore/5.0.0-rc.1.20451.13). `NavigationEntry`, the base class for change tracking entries that track navigation properties now returns `INavigationBase` when retrieving its metadata instead of `INavigation` it used to. Problem is, `INavigationBase` does not have the useful properties `IsOnDependent` or `ForeignKey` that I like to use so often; those are offered by `INavigation` only. As it turns out, the problem is with the return type of `NavigationEntry.Metadata` - the object to be returned is cast to `INavigationBase` while in fact it is of type `INavigation`

```csharp
public new virtual INavigationBase Metadata => 
    (INavigationBase)base.Metadata;
```

The simplest fix is to cast back to `INavigation`. I did not want to visit every place and add a cast there, so instead, I added an extension method `Metadata()` that wraps the property access and performs the cast for me (and also because we can’t have extension properties and even if we could, that would result in a name clash in this case).

``` csharp
public static INavigation Metadata(this NavigationEntry navigationEntry) => 
    (INavigation)navigationEntry.Metadata;
```
Now in every place, I call `NavigationEntry.Metadata`, I changed the call from `Metadata` (the property) to `Metadata()` (the extension method) to get `IsOnDependent` or `ForeignKey` back.

```csharp
... if (!referenceEntry.Metadata().IsOnDependent) ...
```

## Failing tests

With the last of the build errors fixed, it was onwards to tests. At this point, we have close to a thousand backend tests. How many would fail?

### SetUpFixture - “Not a TestFixture, but TestSuite”

At the onset of development when all the big decisions are made (you know, the ones that can only be undone at great cost if you come to change your mind later), we picked [NUnit](https://nunit.org/) as our backend test framework. I’ve been using it since the early days of .NET Framework 2.0 and I always thought it had everything a .NET developer needs from a test framework. I’ve always been particularly fond of its concept of constraints - the way they just flow like a natural language, pure elegance. But I digress…

We use the [\[SetUpFixture\]](https://docs.nunit.org/articles/nunit/writing-tests/attributes/setupfixture.html) attribute in our test assemblies to run an assembly wide test setup. After upgrade, the tests would not even run and kept failing prematurely with _“Not a TestFixture, but TestSuite”_ error. Naturally, I went to GitHub first to see if anybody else came across this issue earlier and found an open [issue 770](https://github.com/nunit/nunit3-vs-adapter/issues/770) which seemed like it could be it. I dug around in the code of [NUnit 3 VS Test Adapter](https://github.com/nunit/nunit3-vs-adapter) (specifically the code that drives how NUnit discovers test fixtures in test assemblies) to see if I can spot any simple workaround, but no luck. Earmarking this for a detailed follow-up later, I decided to take inventory of the affected tests and just commented out the problematic part to see which tests would fail. To my surprise, just a single one. Well, what a way to learn about useless code in your codebase!

### Task - is really a Task\<VoidTaskResult\> 

One of the tests started failing and after a quick inspection, the problem was found not with the system under test but with the code of assertion itself. What previously worked, broke with .NET 5.

```csharp
Assert.That(actual, Is.Not.Null.And.TypeOf<Task>().And.EqualTo(Task.CompletedTask));
```

The async method that used to return `Task` from a void method, is now returning `Task<VoidTaskResult>` where `VoidTaskResult` represents a void return. Unfortunately, `VoidTaskResult` is not a public class so we can’t use it in our test. I can only wonder about the reason for the change, but the framework designers probably saw value in representing the return type void as a generic type parameter. This could, for example, allow `Task<T>` to be the base class of all Tasks, and retroactively make `Task` its subclass (just my humble hypothesis, I have yet to look at the code). Since I don’t need to know the exact type as long as it quacks like a `Task`, the fix is quite simple.

```csharp
Assert.That(actual, Is.Not.Null.And.AssignableTo<Task>().And.EqualTo(Task.CompletedTask));
```

## Handing it over to CI

Once I had a solution that builds locally and has no failing tests, it was time to push it to remote and have our CI in Azure Pipelines take a crack at it. I created a new pipeline based on the YAML we use for out master branch build. The Microsoft-hosted build agent currently comes with .NET 3.1 SDK preset which means I had to add a step that installs the latest .NET SDK to be able to build the solution (make sure to add it before any other steps that invoke dotnet, Nuget, or MSBuild).

```yaml
- task: UseDotNet@2
  displayName: Use latest .NET 5 SDK
  inputs:
    packageType: 'sdk'
    version: '5.x'
    includePreviewVersions: true
```

I pushed my changes to see how it would go on the first try. I got as far as `nuget restore` step, there it went belly-up. 

### Does not restore

The thing is, in our pipeline, before building the solution, we run an explicit `nuget restore` step. I configured it like this a long time ago to get accurate timing of every individual pipeline task to see where we can speed things up. The idea was that if `nuget restore` starts taking up too much time, I’d introduce caching. So far, that wasn’t necessary - right now, the restore step takes about 40 seconds compared to building and restoring cache which takes about twice as much.

The error message that I got from `nuget restore` read:

> ##[error]The nuget command failed with exit code(1) and error(NU1202: Package Microsoft.EntityFrameworkCore.SqlServer 5.0.0-rc.1.20451.13 is not compatible with net50 (.NETFramework,Version=v5.0). Package Microsoft.EntityFrameworkCore.SqlServer 5.0.0-rc.1.20451.13 supports: netstandard2.1 (.NETStandard,Version=v2.1)

Alternatively, if your solution includes any projects targeting .NET Standard, you may get a following error message:

> ##[error]The nuget command failed with exit code(1) and error(NU1201: Project YetAnotherCloudAccounting is not compatible with net50 (.NETFramework,Version=v5.0). Project YetAnotherCloudAccounting supports: netstandard2.1 (.NETStandard,Version=v2.1)

(Don't mind the .NET Framework reference in the error message, that's a known problem already reported in [issue 5820](https://github.com/dotnet/msbuild/issues/5820))

Since I built the solution locally with [Visual Studio 2019 Preview](https://visualstudio.microsoft.com/vs/preview/) (v16.8.0 Preview 5.0) and did not run an explicit `nuget restore` step, I tried running `nuget restore` locally to see if the same issue would reproduce on my machine. The restore command ran to completion without an error. I, therefore, hypothesized that if the packages were restored just fine on my machine, the first place to look is the Nuget version difference between my machine and the hosted build agent. I figured that the Visual Studio 2019 Preview installer would have updated all necessary components including Nuget. And there it was - the version of Nuget on the hosted agent was `5.4.0.6315`, while I was already on `5.7.0.626`. Adding a new task to my pipeline prior to Nuget restore step fixed things right up:

```yaml
- task: NuGetToolInstaller@1 
  displayName: Upgrade Nuget to 5.6.0 (or higher) to restore packages using net5.0 TFM 
  inputs: 
    versionSpec: '>= 5.6.0'
```
I went to Nuget GitHub repo to double-check my solution and indeed the support for net5.0 TFM was added in [issue 9584](https://github.com/NuGet/Home/issues/9584) which was released with Nuget 5.6.0.

Locally, one would achieve the same by running the command below (if you just want the latest, not a specific version)

```powershell
nuget update -Self 
```

## Restores, does not build

Things were starting to look up, only to have the pipeline fail on the very next step. It didn’t compile.

> ##[error]/opt/hostedtoolcache/dotnet/sdk/5.0.100-rc.2.20479.15/Sdks/Microsoft.NET.Sdk/targets/Microsoft.PackageDependencyResolution.targets(241,5): Error NETSDK1005: Assets file '/home/vsts/work/1/s/YetAnotherCloudAccounting/obj/project.assets.json' doesn't have a target for 'net5.0'. Ensure that restore has run and that you have included 'net5.0' in the TargetFrameworks for your project.

When `nuget restore` runs, besides downloading the packages to the local cache, it also generates a list of dependencies for every project and writes it into `project.assets.json` file under project's `/obj` directory. This file is later consumed by the build to resolve the dependencies (basically, for every package reference it will point the build to the local copy of the package). Because I restored the packages in an earlier step, I can run `dotnet build` with `--no-restore` option and skip the restore step during the build. However, both Nuget and dotnet rely on MSBuild to parse and understand the TFM. It would appear, the MSBuild version used by Nuget task on Microsoft hosted agent was quite old as the task’s log shows:

> MSBuild auto-detection: using msbuild version '15.0' from '/usr/lib/mono/msbuild/15.0/bin'. Use option -MSBuildVersion to force nuget to use a specific version of MSBuild.

Compared to my local version 16.8, this felt quite outdated. As the error message suggested, I tried first to force Nuget to use a specific MSBuild version. Unfortunately, this required that we use a custom Nuget task and pass every argument as one string (make sure to reference full path to your solution file, preferably using predefined variables):

```yaml
- task: NuGetCommand@2 
  displayName: Restore packages with NuGet 
  inputs: 
    command: 'custom' arguments: 'restore $(Build.SourcesDirectory)/$(solution) -ConfigFile nuget.config -MSBuildVersion 16.8 -Verbosity detailed'
```
That didn’t quite work out:

> ##[error]The nuget command failed with exit code(1) and error(Cannot find the specified version of msbuild: '16.8'

The MSBuild path on my machine points to the Visual Studio 2019 Preview `/bin` directory. I believe the MSBuild installed together with .NET 5 SDK on the build agent was not installed as a global tool and merely added to PATH. I could have tried referencing it using Nuget's `MSBuildPath` parameter or perhaps even installing it as global tool, but I thought it much simpler to use dotnet CLI to restore the packages. After all, it’s part of the SDK, it should be able to use the SDK’s local MSBuild installation.

```yaml
- task: DotNetCoreCLI@2 
  displayName: Restore packages with dotnet CLI 
  inputs: 
    command: 'restore' 
    projects: '$(solution)' 
    feedsToUse: 'config' 
    nugetConfigPath: 'nuget.config'
```

This worked like a charm and now the solution would restore and build just fine.

## Builds, tests fail

Onto the next problem. Now that I had a solution that built fine it was off to the tests. To my surprise, a large number of tests (~150) were failing. Before investigating them all in detail, I reran the pipeline to see if I would get the same errors or if I was dealing with a more sinister intermittent test failure. 

I was expecting a difficult battle because no tests were failing on my machine and without anything to go by, I would be going around blindly, making educated guesses at best. What I got were different tests failing on each run. Most of them seemed to share one thing in common - an IO failure mentioning buffer read. This led me to suspect either [HttpClient](https://docs.microsoft.com/en-us/dotnet/api/system.net.http.httpclient) or [HttpResponseMessage](https://docs.microsoft.com/en-us/dotnet/api/system.net.http.httpresponsemessage) were not being used properly. Or perhaps the [WebApplicationFactory](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.testing.webapplicationfactory-1) might have changed and caused this. 
Starting with the most likely suspects, I rolled up my sleeves and took the time to refactor our code and make sure we were properly creating, sharing, reusing, and disposing instances of `HttpClient` and `HttpResponseMessage`. It didn’t help, but we ended up with a slightly cleaner codebase. Before going after the next suspect, I went to check dotnet runtime repository on GitHub to see if there were any recent changes to these classes in .NET 5 that could point me to a solution (or perhaps an issue was already filed). Not finding anything, I decided to leave it as-is for a day or two until I can come up with a better hypothesis.

When I returned after a couple of days to take another stab at it, I started by re-running the pipeline to get a fresh set of failed tests to look at. To my surprise, all tests were now green and the pipeline ran to completion. While I was happy to see that, one can only trust the code that one can explain. The only possible explanation was it had to come from somewhere else. I remembered I created the pipeline with a task that installs the _latest_ SDK meaning the most likely culprit that I could think of was a new .NET SDK. Sure enough, one day before, a new SDK (.NET 5.0 RC2) came out as evidenced by the pipeline logs:

> Tool to install: .NET Core sdk version 5.x.  
> Found version 5.0.100-rc.2.20479.15 in channel 5.0 for user specified version spec: 5.x  
> Version 5.0.100-rc.2.20479.15 was not found in cache.  
> Getting URL to download .NET Core sdk version: 5.0.100-rc.2.20479.15.  
> Detecting OS platform to find correct download package for the OS.  

 I reran the pipeline multiple times to confirm it was now stable and working fine. EVerything was OK now.