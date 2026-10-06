# pra-infra

The platform's network path between clusters, for the E5 environment.

Services in `e5-client` reach services in `e5-server` through an
**appliance**: a stock Envoy running on the shared network between the
clusters (`e5/envoy.yaml`). Clients do not know about it; they dial a name.

- `e5/dnsmasq.conf` — the programmed DNS: `api.e5.internal` answers with the
  appliance's address. `e5/coredns.Corefile` forwards the `e5.internal`
  zone from the client cluster's CoreDNS to it.
- `e5/envoy.yaml` — the appliance: listener `:8443`, a **local rate limit
  of 20 requests per second** (token bucket), a **retry policy of 3**
  toward the server, circuit breakers, and admin stats on `:9901`.
- `e5/server-exposure.yaml` — the server side: `hello` exposed on NodePort
  `30051` of the server cluster's nodes, which the appliance forwards to.
- `e5/prometheus.yaml` — one Prometheus in the client cluster scraping both
  clusters' state and the appliance's stats (`component="appliance"`).

Limits of the path, as configured:

| Hop | Limit |
|---|---|
| appliance ingress | 20 rps, then `429` (`envoy_http_local_rate_limit_rate_limited`) |
| appliance → server | 3 retries on unavailable/connect-failure; 200 in-flight; 64 connections |
| server | 2 replicas, no autoscaler |
