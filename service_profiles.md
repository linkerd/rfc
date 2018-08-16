# Service Profiles

The RFC introduces the notion of service profiles.  A service profile is an
object that describes characteristics that a service may have.  Throughout
this RFC we will extensively use the REST Service Profile as an example.  The
REST Service Profile describes the characteristics of an HTTP service that
obeys the tenants of [REST](https://en.wikipedia.org/wiki/Representational_state_transfer).

Service profiles are useful because this allows Linkerd to behave differently
depending on the characteristics of the service to which it is making requests.
For example, if Linkerd sends an HTTP GET request to a service that has the
REST profile, the request is guaranteed to be idempotent and therefore is safe
for Linkerd to retry.  The types of characteristics that could potentially be
described by a service profile includes:

* success classification
* retryability/idempotence
* retry budgets
* rate limits
* ACLs
* latency objectives/timeouts
* success rate objectives
* etc.

Each service can be associated with at most one profile.  While we may one day
wish to explore extending or overriding profiles, this is currently out of
scope.  It is important to note that even though it is a service that is
associated with a profile, this can influence the behavior of callers of that
service.  In many cases the service and its caller may exist in different
namespaces and/or have different owners.

Service profiles are designed to be built in an incremental and extensible way.
The initial implementation will contain very few characteristics but will be
expanded over time.

## Service Profile Definition

While some characteristics such as retry budgets apply to the service as a
whole, many characteristics vary depending on the request.  For example,
idempotence may vary depending on the request's HTTP method or timeouts might
vary depending on the request's HTTP path.

To accommodate this, a service profile contains some top level characteristics
that apply to the whole service and an ordered list of request policy objects
which define the characteristics which apply to certain classes of requests.

```
message ServiceProfile {
  repeated RequestPolicy request_policies = 1;
  RetryBudget retry_budget = 2;
  Duration default_timeout = 3;
  // More service level characteristics to be added in the future
}
```

Each request policy object defines a request match condition that describes
which requests fall under this policy.  If a request matches more than one
request policy object's condition, only the first match is considered.  The
request policy object also defines the characteristics that apply to its
requests.  For example, it defines a response match condition that describes
which responses should be considered successful and another response match
condition that describes which responses should be considered retryable if they
are not successful.

```
message RequestPolicy {
  RequestMatch condition = 1;
  ResponseMatch success_condition = 2;
  ResponseMatch retry_condition = 3;
  Duration timeout = 4;
  // More request policy characteristics to be added in the future
}
```

For example, consider the HTTP REST service profile:

```
ServiceProfile {
  request_policies: [
    RequestPolicy {
      // idempotent requests
      condition: Any(Method(GET), Method(PUT), Method(DELETE), Method(HEAD), Method(TRACE))
      success_condition: Status < 500
      retry_condition: Always
    },
    RequestPolicy {
      // fallback, (non-idempotent)
      condition: Always
      success_condition: Status < 500
      retry_condition: Never
    }
  ],
  retry_budget: RetryBudget(0.2),
  default_timeout: 6 seconds
}
```

## Further Work

In this example we have given a simple service profile that describes what
constitutes a successful response for an HTTP REST service and which types of
requests are safe to retry.  Many other profiles are also possible, such as a
gRPC profile that defines a successful response as one with grpc-status trailer
with value 0.  Ultimately, users should be able to define service profiles
specific to their service which describe the exact characteristics of their
service including which paths are retryable.

This framework enables a wide range of features which require information about
services.  A few examples include setting timeouts based on latency objectives,
doing speculative retries based on retryability, enforcing rate limits,
requiring identity (mutual auth TLS), and much more.