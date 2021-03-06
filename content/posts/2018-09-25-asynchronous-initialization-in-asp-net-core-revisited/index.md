---
layout: post
title: Asynchronous initialization in ASP.NET Core, revisited
date: 2018-09-25T00:00:00.0000000
url: /2018/09/25/asynchronous-initialization-in-asp-net-core-revisited/
tags:
  - asp.net core
  - async
  - C#
  - initialization
categories:
  - ASP.NET Core
---


Initialization in ASP.NET Core is a bit awkward. There are well defined places for registering services (the `Startup.ConfigureServices` method) and for building the middleware pipeline (the `Startup.Configure` method), but not for performing other initialization steps (e.g. pre-loading data, seeding a database, etc.).

## Using a middleware: not such a good idea

Two months ago I published [a blog post about asynchronous initialization of an ASP.NET Core app using a custom middleware](/2018/07/20/asynchronous-initialization-in-asp-net-core-with-custom-middleware/). At the time I was rather pleased with my solution, but a comment from Frantisek made me realize it wasn't such a good approach. Using a middleware for this has a major drawback: even though the initialization will only be performed once, the app will still incur the cost of calling an additional middleware for every single request. Obviously, we don't want the initialization to impact performance for the whole lifetime of the app, so it shouldn't be done in the request processing pipeline.

## A better approach: the `Program.Main` method

There's a piece of all ASP.NET Core apps that's often overlooked, because it's generated by a template and we rarely need to touch it: the `Program` class. It typically looks like this:

```csharp
public class Program
{
    public static void Main(string[] args)
    {
        CreateWebHostBuilder(args).Build().Run();
    }

    public static IWebHostBuilder CreateWebHostBuilder(string[] args) =>
        WebHost.CreateDefaultBuilder(args)
            .UseStartup<Startup>();
}
```

Basically, it builds a web host and immediately runs it. However, there's nothing to prevent us from doing something with the host before running it. In fact, it's a pretty good place to perform the app initialization:

```csharp
    public static void Main(string[] args)
    {
        var host = CreateWebHostBuilder(args).Build();
        /* Perform initialization here */
        host.Run();
    }
```

As a bonus, the web host exposes a service provider (`host.Services`), configured with the services registered in `Startup.ConfigureServices`, which gives us access to everything we might need to initialize the app.

But wait, didn't I mention *asynchronous* initialization in the title? Well, since C# 7.1, it's possible to make the `Main` method async. To enable it, just set the `LangVersion` property to 7.1 or later in your project (or `latest` if you always want the most recent features).

## Wrapping up

While we could just resolve services from the service provider and call them directly in the `Main` method, it wouldn't be very clean. Instead, it would be better to have an initializer class that receives the services it needs via dependency injection. This class would be registered in `Startup.ConfigureServices` and called from the `Main` method.

After using this approach in two different projects, I put together a small library to make things easier: [AspNetCore.AsyncInitialization](https://github.com/thomaslevesque/AspNetCore.AsyncInitialization/). It can be used like this:

1. Create a class that implements the `IAsyncInitializer` interface:

```csharp
public class MyAppInitializer : IAsyncInitializer
{
    public MyAppInitializer(IFoo foo, IBar bar)
    {
        ...
    }

    public async Task InitializeAsync()
    {
        // Initialization code here
    }
}
```
2. Register the initializer in `Startup.ConfigureServices`, using the `AddAsyncInitializer` extension method:

```csharp
services.AddAsyncInitializer<MyAppInitializer>();
```
    It's possible to register multiple initializers.
3. Call the `InitAsync` extension method on the web host in the `Main` method:

```csharp
public static async Task Main(string[] args)
{
    var host = CreateWebHostBuilder(args).Build();
    await host.InitAsync();
    host.Run();
}
```
    This will run all registered initializers.


There you have it, a nice and clean way to initialize your app. Enjoy!

