## Rate Limiting with the RateLimitService

Occasionally, your services may become overwhelmed with too many requests. In this situation, global rate limiting is a good solution to prevent cascade failure. Ambassador supports rate limiting via an external third party service. This rate limiting is based on [Envoy Proxy's rate limiting capabilities](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/global_rate_limiting.html).

A `RateLimitService` manifest configures Ambassador to use an external service to check and enforce rate limits for incoming requests:

```yaml
---
apiVersion: ambassador/v0
kind: RateLimitService
name: ratelimit
service: "example-rate-limit:5000"
```

- `service` gives the URL of the rate limit service.

You may only use a single `RateLimitService` manifest.

### The external rate limiting service

When using an external rate limit service, Ambassador will generate a gRPC request to the external rate limit service and will provide a list of descriptors on which the rate limit service can base its decision to accept or reject the request:

```
[
  {"source_cluster", "<local service cluster>"},
  {"destination_cluster", "<routed target cluster>"},
  {"remote_address", "<trusted address from x-forwarded-for>"},
  {"generic_key", "<descriptor_value>"},
  {"<some_request_header>", "<header_value_queried_from_header>"}
]
```

This gRPC service must implement the Envoy [ratelimit.proto](https://github.com/datawire/ambassador/blob/master/ambassador/common/ratelimit/ratelimit.proto).

If Ambassador cannot contact the rate limit service, it will allow the request to be processed as if there were no rate limit service configuration.

It is the external rate limit service's responsibility to determine whether rate limiting should take place, depending on custom business logic. The rate limit service must simply respond to the request with an `OK` or `OVER_LIMIT` code:
* If Envoy receives an `OK` response from the rate limit service, then Ambassador allows the client request to resume being processed by the normal Ambassador Envoy flow.
* If Ambassador receives an `OVER_LIMIT` response, then Ambassador will return an HTTP 429 response to the client and will end the transaction flow, preventing the request from reaching the backing service.

The headers injected by the [AuthService](auth-service.md) can also be passed to the rate limit service since the `AuthService` is invoked before the `RateLimitService`.

## Example

See [the Ambassador Rate Limiting Tutorial](../../user-guide/rate-limiting-tutorial.md) for an example.
