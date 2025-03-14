Monitoring the health of applications is vital when running them in containerized environments (e.g., Kubernetes), integrating with status pages, and leveraging observability tools like New Relic. This document explains the three primary types of health checks that will be implemented in our APIs built with Microsoft .NET 8, and it outlines best practices on execution frequency and the depth of checks.

## Liveness

A Liveness probe verifies that the application is running and not in a deadlocked or unresponsive state. It answers the question, “Is the application alive?” Typically, it is a lightweight check that does not inspect external dependencies.

When to Use It
- Post-Startup: Once the application has successfully started, use liveness checks to periodically confirm that it remains responsive.
- Long-Running Processes: For applications that may encounter deadlocks, memory leaks, or similar issues that render them nonfunctional without crashing.

Why to Use It
- Automatic Recovery: If the liveness check fails, orchestrators like Kubernetes can automatically restart the container.
- Minimal Overhead: A simple check that confirms the basic viability of the application without incurring heavy performance costs.

How to Use It

Implement a liveness endpoint that returns a healthy response when the application is operating normally. Below is an example using .NET 8 minimal API standards in C#:

```csharp
using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Microsoft.Extensions.Diagnostics.HealthChecks;

var builder = WebApplication.CreateBuilder(args);

// Register health checks with a specific tag for liveness
builder.Services.AddHealthChecks()
    .AddCheck("Liveness", () => HealthCheckResult.Healthy("The app is alive."), tags: new [] { "liveness" });

var app = builder.Build();

// Map the liveness endpoint
app.MapHealthChecks("/healthz/ping", new HealthCheckOptions {
    Predicate = (check) => check.Tags.Contains("liveness")
});

app.Run();
```

This endpoint allows Kubernetes or monitoring tools to quickly verify that the application process is active.

## Readiness

A Readiness probe determines whether the application is ready to handle incoming requests. It typically verifies that all dependent services (e.g., databases, external APIs) are available in addition to confirming basic application health.

When to Use It
- Before Routing Traffic: Use readiness checks during startup to delay sending traffic until all dependencies are available.
- Ongoing Operations: Continually validate that the application can handle requests, especially following configuration changes or dependency issues.

Why to Use It
- Load Balancer Integration: Ensures that only healthy, ready instances receive traffic, which enhances overall system reliability.
- Dependency Verification: Helps detect issues in external integrations that could impair request handling even if the application process is running.

How to Use It

Set up a readiness endpoint that performs deeper checks, including external dependency tests. Here’s an example in C# using .NET 8:

```csharp
using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Microsoft.Extensions.Diagnostics.HealthChecks;

var builder = WebApplication.CreateBuilder(args);

// Register health checks with a readiness tag
builder.Services.AddHealthChecks()
    .AddCheck("Readiness", () =>
    {
        // Insert custom readiness logic here (e.g., database connectivity check)
        bool dependenciesHealthy = true; // Replace with real check
        return dependenciesHealthy 
            ? HealthCheckResult.Healthy("The app is ready.")
            : HealthCheckResult.Unhealthy("A dependency is failing.");
    }, tags: new [] { "readiness" });

var app = builder.Build();

// Map the readiness endpoint
app.MapHealthChecks("/healthz/ready", new HealthCheckOptions {
    Predicate = (check) => check.Tags.Contains("readiness")
});

app.Run();
```

This endpoint informs orchestrators or load balancers that the application is fully prepared to process requests.

## Startup

A Startup probe confirms that the application has successfully completed its initialization routines. This check distinguishes between an application that is still starting up and one that has fully started but may encounter runtime issues later.

When to Use It
- During Initialization: Use the startup check as part of the bootstrapping process to determine if the application has finished all startup tasks (e.g., loading configurations, establishing connections, running migrations).
- Delayed Readiness: Prevents traffic from being routed to an instance that isn’t fully initialized.

Why to Use It
- Separation of Concerns: Differentiates between startup-related issues and runtime failures, providing clearer troubleshooting signals.
- Graceful Startup: Allows orchestrators like Kubernetes to manage startup time independently, ensuring containers are not prematurely terminated.

How to Use It

Implement a startup endpoint that focuses on initialization checks. Here’s an example using .NET 8 and C#:

```csharp
using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Microsoft.Extensions.Diagnostics.HealthChecks;

var builder = WebApplication.CreateBuilder(args);

// Register health checks with a startup tag
builder.Services.AddHealthChecks()
    .AddCheck("Startup", () =>
    {
        // Insert custom startup logic here (e.g., configuration load or migration status)
        bool initializationComplete = true; // Replace with real startup check logic
        return initializationComplete 
            ? HealthCheckResult.Healthy("Startup completed successfully.")
            : HealthCheckResult.Unhealthy("Startup is still in progress or failed.");
    }, tags: new [] { "startup" });

var app = builder.Build();

// Map the startup endpoint
app.MapHealthChecks("/healthz/startup", new HealthCheckOptions {
    Predicate = (check) => check.Tags.Contains("startup")
});

app.Run();
```

This startup endpoint ensures that the application signals a healthy initialization before being marked as ready for live traffic.

## Best Practices

Frequency of Health Checks
- Liveness Checks:
	- Frequency: These should be executed frequently (e.g., every 10–30 seconds) because they are designed to quickly detect if the application has become unresponsive.
	- Rationale: Since liveness checks are lightweight, frequent polling helps in rapidly identifying issues without imposing significant overhead.
- Readiness Checks:
	- Frequency: Consider running these every 30 seconds to 1 minute.
	- Rationale: Because readiness checks might involve verifying external dependencies, a slightly lower frequency can reduce the performance impact. If some checks are particularly heavy, consider caching their results for a short period.
- Startup Checks:
	- Frequency: These are typically used only during the initialization phase.
	- Rationale: Once startup is complete, continuous rechecking is unnecessary. Focus on ensuring the application transitions to a ready state before accepting traffic.

Depth of Health Checks
- Keep Checks Lightweight:
	- Scope: Health checks should perform shallow validations (e.g., basic connectivity or quick status assessments) rather than full-scale integration tests.
	- Rationale: Deep checks can delay response times and might inadvertently lead to cascading failures if they depend on several external systems.
- Direct Dependency Checks Over Chained Endpoints:
	- Inter-Service Dependencies:
		- Best Practice: If API A depends on API B, it is not recommended for API A to query API B’s readiness endpoint directly.
		- Alternative: Instead, API A should perform a minimal connectivity or ping check to API B, or rely on its own direct integration test if necessary. This avoids propagating the health check load across services and prevents potential chaining issues.
- Avoid Overloading Health Check Endpoints:
	- Design: Ensure that health check endpoints only perform the necessary checks to determine service status, rather than executing complete business logic or transactions.
	- Example: Instead of triggering full database queries, a readiness check might simply verify that a database connection can be established.

## Conclusion

Implementing these health check probes with best practices in frequency and depth allows us to:
- Quickly Detect Issues: Whether during runtime (liveness), in handling requests (readiness), or during initialization (startup).
- Enhance Reliability: Kubernetes, status pages, and monitoring tools like New Relic can proactively manage traffic and recover from failures based on these checks.
- Optimize Performance: By carefully choosing the execution frequency and ensuring that checks are shallow, we avoid potential delays caused by testing multiple dependencies.
- Prevent Cascading Failures: Avoid chaining health check endpoints across services. Instead, use lightweight direct checks or rely on orchestration mechanisms for managing inter-service dependencies.