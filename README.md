# BackgroundTasks_Hangfire_NET8

Understanding BackgroundTasks in .Net 8 C#

https://medium.com/c-sharp-programming/understanding-backgroundtasks-in-net-8-c-19bedb56fc7b

Using Hangfire in .NET 8 for Background Jobs

https://medium.com/@valentin.osidach/using-hangfire-background-jobs-in-net-8-95c0ac30c1ec

## Setting Up a Background Task
```
using System;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

public class MyBackgroundService : BackgroundService
{
    private readonly ILogger<MyBackgroundService> _logger;

    public MyBackgroundService(ILogger<MyBackgroundService> logger)
    {
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Background task starting.");

        while (!stoppingToken.IsCancellationRequested)
        {
            _logger.LogInformation("Background task doing work.");
            //Your code here ...
            await Task.Delay(1000, stoppingToken); 
        }

        _logger.LogInformation("Background task stopping.");
    }
}
```

## Registering the Background Service
```
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHostedService<MyBackgroundService>();
```

# Advanced Scenarios with BackgroundTasks

+ **Scoped Services**: If your background task needs to interact with services registered with a scoped lifetime, you can create a scope inside your task.
+ **Periodic Tasks**: You can leverage timers or scheduling libraries for tasks that need to run at regular intervals.
+ **Concurrency**: If your task involves parallel operations, ensure proper concurrency management to avoid issues like race conditions.

## Background Task with Scoped Services
```
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using System;
using System.Threading;
using System.Threading.Tasks;

public class ScopedBackgroundService : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<ScopedBackgroundService> _logger;

    public ScopedBackgroundService(IServiceProvider serviceProvider, ILogger<ScopedBackgroundService> logger)
    {
        _serviceProvider = serviceProvider;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Scoped Background Service is starting.");

        while (!stoppingToken.IsCancellationRequested)
        {
            _logger.LogInformation("Scoped Background Service is doing background work.");

            // Create a new scope
            using (var scope = _serviceProvider.CreateScope())
            {
                //Your service
                var scopedProcessingService = scope.ServiceProvider.GetRequiredService<IScopedProcessingService>();

                await scopedProcessingService.DoWork(stoppingToken);
            }

            await Task.Delay(5000, stoppingToken); // Simulate some delay between work cycles
        }

        _logger.LogInformation("Scoped Background Service is stopping.");
    }
}
```

### Registering the Background Service
```
var builder = WebApplication.CreateBuilder(args);
var services = builder.Services;

services.AddHostedService<MyBackgroundService>();
services.AddScoped<IScopedProcessingService, ScopedProcessingService>();
```

# Using Hangfire in .NET 8 for Background Jobs
```
dotnet new webapi -n HangfireDemo
dotnet add package Hangfire
dotnet add package Hangfire.AspNetCore
```

## Configure Hangfire
```
using Hangfire;
using Hangfire.MemoryStorage;

var builder = WebApplication.CreateBuilder(args);

...

builder.Services.AddHangfire(config =>
{
    config.UseMemoryStorage(); // In-memory storage for simplicity; replace with a persistent storage in production
});
builder.Services.AddHangfireServer(); 

var app = builder.Build();

...

app.UseHangfireDashboard();

app.Run();
```

# Hangfire supports different types of background jobs:

+ **Fire-and-forget jobs**: These are executed once, immediately, or after a delay.
+ **Recurring jobs**: These are executed on a schedule.
+ **Delayed jobs**: These are executed after a specified delay.
+ **Continuation jobs**+ : These are executed after a parent job finishes.

## Fire-and-forget examples
```
--Simple background job
BackgroundJob.Enqueue(() => Console.WriteLine("Background job executed!"));

--Background job with DI
BackgroundJob.Enqueue<IBackgroundService>(service => service.RunAsync());
```

## Recurring Jobs
```
RecurringJob.AddOrUpdate(() => Console.WriteLine("Recurring job executed!"), Cron.Daily);
```

### Example with DI
```
interface IRecurringJob
{
    // Cron interval 
    // Example "5 4 * * *" equals “Every Day At 04:05.” 
    string CronInterval { get; }
    Task Start(PerformContext? context);
}

abstract class RecurringJob : IRecurringJob
{
    public abstract string CronInterval { get; }

    public abstract Task Start(PerformContext? context);
}
```

### SimpleJob
```
public class SimpleJob : RecurringJob
{
    //Run job every hour
    public override string CronInterval => "0 * * * *"; 

    public override async Task Start(PerformContext? context)
    {
        await Task.CompletedTask;
    }
}

public class SimpleJobWithDIJob(IService service) : RecurringJob
{
    //Run job every hour
    public override string CronInterval => "0 * * * *";

    public override async Task Start(PerformContext? context)
    {
        await service.ExecuteAsync();
    }
}
```

### Job initializer
```
public static class RecurringInitializer
{ 
    public static void InitializeRecuringJobs(this IServiceProvider serviceProvider)
    {
        var jobs = serviceProvider.GetServices<IRecurringJob>();

        foreach (var job in jobs)
        {
             var jobId = job.GetType().Name;

             RecurringJob.AddOrUpdate(jobId, () => job.Start(default), job.CronInterval);
        }
   }
}
```

## Register jobs in the Program
```
services.AddScoped<IHangfireJob, SimpleJob>();
services.AddScoped<IHangfireJob, SimpleJobWithDIJob>();


services.InitializeRecuringJobs();
```
