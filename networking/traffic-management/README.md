# Traffic management

Steering and shaping application traffic: proxies, load balancing at layers 4 and 7, service discovery, content delivery, service meshes and the nginx server.

## Map

The diagram shows the notes in reading order; subareas are marked with (subarea).

```mermaid
flowchart LR
  n0["Proxies"]
  n1["Reverse proxy"]
  n0 --> n1
  n2["Forward proxy"]
  n1 --> n2
  n3["Load balancing"]
  n2 --> n3
  n4["Load balancing algorithms"]
  n3 --> n4
  n5["Layer 7 load balancing"]
  n4 --> n5
  n6["Layer 4 load balancing"]
  n5 --> n6
  n7["Service discovery"]
  n6 --> n7
  n8["CDN"]
  n7 --> n8
  n9["Service mesh"]
  n8 --> n9
  n10["nginx"]
  n9 --> n10
```

## Notes in reading order

1. [Proxies](proxies.md): definition and the types of proxies.
2. [Reverse proxy](reverse-proxy.md): benefits and comparison with load balancers.
3. [Forward proxy](forward-proxy.md): egress control, filtering and DevOps uses.
4. [Load balancing](load-balancing.md): benefits, use cases and horizontal scaling.
5. [Load balancing algorithms](load-balancing-algorithms.md): static, dynamic and consistent hashing.
6. [Layer 7 load balancing](layer-7-load-balancing.md): content-based routing and features.
7. [Layer 4 load balancing](layer-4-load-balancing.md): forwarding modes and use cases.
8. [Service discovery](service-discovery.md): registries, client-side and server-side discovery.
9. [CDN](cdn.md): edge caching, push and pull, and how requests are routed.
10. [Service mesh](service-mesh.md): data and control planes, mTLS and traffic policies.
11. [nginx](nginx.md): architecture, reloads and configuration.

## Planned topics

- HAProxy
- Envoy
- API gateways
- Rate limiting and traffic shaping
- Global server load balancing and anycast
