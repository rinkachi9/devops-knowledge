# Containers and virtualization

Running workloads in isolated environments: hypervisors and virtual machines, and containers with Docker.

Kubernetes is covered in the separate Kubernetes knowledge repository (`kubernetes-knowledge`).

## Map of the area

The diagram shows the notes in reading order; subareas are marked with (subarea).

```mermaid
flowchart LR
  n0["Virtualization"]
  n1["Docker"]
  n0 --> n1
```

## Notes in reading order

1. [Virtualization](virtualization.md): hypervisors, architectures, kinds of virtualization and comparison with containers.
2. [Docker](docker.md): images, containers, registries, volumes, networks, Compose and internals.

## Planned topics

- Container internals: namespaces, cgroups and overlayfs
- OCI images and registries
- Container runtimes: containerd, CRI-O, runc
- Podman and rootless containers
- Container image security and scanning
- Building small images
