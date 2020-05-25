---
title: "Primer: Designing a health-check API"
date: 2017-12-29T02:01:58+05:30
tags: [healthcheck, microservices]
---

If you are working on a web application, be it a monolith, micro-service or even serverless architecture, you should be able to find out very easily whether your application is healthy or not, and if it isn't, then it shouldn't be serving any traffic at all. Implementing a simple health-check API will allow you to do just that.

## Why do we need it?

It informs the load balancer distributing traffic to your application or a service discovery (a registry that among other things may periodically checks health of the application hosts) that your application running on a specific host will be unable to serve traffic when it's unhealthy. If found unhealthy, a load balancer may take that host running the application out of rotation, or in case of a service discovery, web service query using DNS will omit the unhealthy host.
In case of some weird behaviour with the application on a specific host, you could take that host out of rotation manually, so that you can debug the application without having to shut down the process.

## Design Considerations

Always check if any of your core application dependencies are failing and return an HTTP status code of 503 (Service Unavailable) along with a list of errors raised by the failing dependencies. 
Eg: connection to a database, invalid S3 credential, low disk space (if your application saves files temporarily to disk).
Health-check endpoints are called frequently and within fixed intervals, therefore your latencies should ideally be small. Enforce an overall upper bound for your health-check response and make sure to cancel any pending checks to the downstream dependencies in case of timeout.

If you are performing multiple dependency checks over the network, try to process them concurrently, if possible, to reduce the overall response time.

As mentioned previously, providing the ability to toggle traffic for a specific host can help in debugging the app on the host without having to shut the process down. 

Eg: A simple file existence check on the host is one traditional approach.
By default, any core dependency error leads to a 503 HTTP status code. But certain dependencies failures may not completely make our application non-operational(e.g. metrics service, external service failures, etc.). For such failures, we should be able to simply observe the errors in the API response without having to take the host out of rotation, but still be captured in logs for alerts.

---

I created a simple library that returns an HTTP handler implementing the health-check API for Go based web services.

The library defines a Checker interface that takes in a context.Context object which is used for signalling request cancellation to the checker functions in case of timeout/request cancellation and returns an error in case if any were detected.
You can define your own custom implementation of the checker and inject it while creating the handler.

```go
type Checker interface {
    Check(ctx context.Context) error
}
type CheckerFunc func(ctx context.Context) error
```

The library uses functional options to accept optional arguments during initiation of the HTTP Handler for overall timeout, observers and custom checkers.

```go
// For the sake of brevity, error check is being omitted here.
db, _ := sql.Open("mysql", "user:password@/dbname")
defer db.Close()
r := mux.NewRouter()
r.Handle("/healthcheck", healthcheck.Handler(
    // WithTimeout option allows you to set a max overall timeout
    healthcheck.WithTimeout(5*time.Second),
    // Checkers fail the status in case of any error.
    healthcheck.WithChecker(
        "database", healthcheck.CheckerFunc(
            func(ctx context.Context) error {
                return db.PingContext(ctx)
            },
        ),
    ),
    // Observers do not fail the status in case of error.
    healthcheck.WithObserver(
        "diskspace", checkers.DiskSpace("/var/log", 90),
    ),
))
```

In case of an error from the database, you should see the following response with the status code 503.

```json
{
  "status": "Service Unavailable",
  "errors": {
    "database": "dial tcp 127.0.0.1:3306: getsockopt: connection refused"
  }
}
```

A more complete example is provided in the [GitHub page](https://github.com/etherlabsio/healthcheck). Please do check it out and help us make it better with your contributions.
