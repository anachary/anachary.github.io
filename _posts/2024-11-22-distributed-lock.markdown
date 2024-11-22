--- 
layout: post
title:  "Running Background Jobs in Containerized Apps"
summary: "Running Background Jobs in Containerized Apps"
author: anachary
date: '2024-11-22 14:35:23 +0530'
category: "cloud"
thumbnail: /assets/img/posts/background-job-thumbnail.png
keywords: logical programmer, Kubernetes,Docker,Dotnet"
permalink: /blog/2024-11-22-distributed-lock/
usemathjax: true
---

# Running Background Jobs in Containerized Apps

Ensuring that a critical background task executes reliably and efficiently in a distributed environment is a complex challenge. A common pitfall is allowing multiple instances of the same task to run simultaneously, which can lead to data inconsistencies, resource contention, and unexpected behavior.

The Scenario:

I recently faced this challenge while developing a highly scalable microservice application built with .NET Core 8.0 and deployed to Azure Kubernetes Service (AKS). The background service, scheduled to run hourly, required a strict guarantee that only one instance was active at a time, regardless of the number of deployed replicas.

The Solution:

To address this challenge, I explored several approaches and ultimately implemented a robust solution. In this post, I'll delve into the various strategies I tested, the challenges I encountered, and the final solution that ensured the reliability and integrity of my background service.
## The Challenge

Here was my situation:
- A .NET Core web application running in Docker containers
- A background job that needs to run every hour
- Multiple replicas running in AKS clusters
- Need to ensure the job runs in exactly one container

## Attempt #1: The Naive Approach - Just Add a Background Service

My first thought was simple - just create a BackgroundService in .NET Core. Here's what I tried:

```csharp
public class HourlyJobService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await DoWork();
            await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
        }
    }
}
```

The problem? It ran in every container! 😅 Multiple jobs running simultaneously - definitely not what we wanted.

## Attempt #2: Distributed Lock with Redis

Next, I thought - "Aha! I'll use Redis to implement a distributed lock!"

```csharp
public class HourlyJobService : BackgroundService
{
    private readonly IDistributedLockManager _lockManager;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var lockAcquired = await _lockManager.TryAcquireLockAsync("hourly-job-lock", TimeSpan.FromMinutes(5));
            
            if (lockAcquired)
            {
                try
                {
                    await DoWork();
                }
                finally
                {
                    await _lockManager.ReleaseLockAsync("hourly-job-lock");
                }
            }
            
            await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
        }
    }
}
```

This worked better, but had some issues:
- What if the process crashed while holding the lock?
- Redis became a single point of failure
- Lock duration needed careful tuning

## Attempt #3: Leader Election with Kubernetes

This is where things got interesting. Kubernetes has a built-in leader election mechanism using ConfigMaps or Leases. Here's how I implemented it:

```csharp
public class KubernetesLeaderElectionService : BackgroundService
{
    private readonly IKubernetes _kubernetes;
    private readonly string _podName;
    private readonly string _namespace;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        var leaseClient = _kubernetes.CoordinationV1Namespaced(_namespace);
        
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                // Try to acquire or renew the lease
                var lease = await leaseClient.CreateOrUpdateLeaseAsync(
                    "background-job-lease",
                    new V1Lease
                    {
                        Spec = new V1LeaseSpec
                        {
                            HolderIdentity = _podName,
                            LeaseDurationSeconds = 15
                        }
                    },
                    stoppingToken);

                if (lease.Spec.HolderIdentity == _podName)
                {
                    // We are the leader, do the work
                    await DoWork();
                }
            }
            catch (Exception ex)
            {
                // Handle errors, maybe we lost leadership
                _logger.LogError(ex, "Error during leader election");
            }
            
            await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
        }
    }
}
```

## The Final Solution: Combining Leader Election with Background Jobs

After some iterations, I settled on a more robust solution that combines leader election with proper job scheduling:

```csharp
public class LeaderAwareJobScheduler : BackgroundService
{
    private readonly IKubernetes _kubernetes;
    private readonly ILogger<LeaderAwareJobScheduler> _logger;
    private readonly IJobExecutor _jobExecutor;
    private bool _isLeader;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // Start leader election process
        _ = RunLeaderElectionAsync(stoppingToken);
        
        // Schedule jobs only when leader
        var timer = new PeriodicTimer(TimeSpan.FromHours(1));
        
        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            if (_isLeader)
            {
                try
                {
                    await _jobExecutor.ExecuteAsync(stoppingToken);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Error executing scheduled job");
                }
            }
        }
    }
    
    private async Task RunLeaderElectionAsync(CancellationToken stoppingToken)
    {
        var leaderElector = new LeaderElector(_kubernetes, "background-jobs");
        
        leaderElector.OnStartedLeading += () =>
        {
            _isLeader = true;
            _logger.LogInformation("Started leading");
            return Task.CompletedTask;
        };
        
        leaderElector.OnStoppedLeading += () =>
        {
            _isLeader = false;
            _logger.LogInformation("Stopped leading");
            return Task.CompletedTask;
        };
        
        await leaderElector.RunAsync(stoppingToken);
    }
}
```

### Why This Solution Works Best

1. **Native Kubernetes Integration**: Uses Kubernetes' built-in features rather than introducing external dependencies

2. **Fault Tolerance**: 
   - If the leader crashes, a new leader is automatically elected
   - No orphaned locks to deal with
   - Seamless container recreation

3. **Scalability**: Works the same whether you have 2 or 20 replicas

4. **Monitoring**: Easy to track leadership changes through logs

## Deployment Configuration

Here's the Kubernetes deployment configuration I used:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: web-app
        image: your-image:tag
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
```

## Lessons Learned

1. **Start Simple**: While the final solution might look complex, starting simple helped understand the real requirements

2. **Use Platform Features**: Kubernetes provides many building blocks - use them!

3. **Consider Failure Modes**: Always think about what happens when things go wrong

4. **Monitor and Log**: Make sure you can debug issues in production

## Next Steps

If you're implementing something similar, consider:
- Adding metrics for job execution
- Implementing retry policies
- Setting up alerts for leadership changes
- Adding health checks

Remember, there's no one-size-fits-all solution. Your specific needs might require a different approach, but I hope this journey helps you make an informed decision!

Happy coding! 🚀
