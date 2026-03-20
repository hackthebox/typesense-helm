# Changelog

## 1.1.0 ![AppVersion: 30.1](https://img.shields.io/static/v1?label=AppVersion&message=30.1&color=success&logo=) ![Helm: v3](https://img.shields.io/static/v1?label=Helm&message=v3&color=informational&logo=helm)

**Release date:** 2026-03-20

### Added

- PodDisruptionBudget is now enabled by default for multi-node clusters (replicaCount > 1), automatically skipped for single-replica deployments
- Auto-calculated `maxUnavailable` for both PDB and StatefulSet `updateStrategy`, based on Raft fault tolerance (`floor(replicaCount/2)`), with user override support and quorum validation
- Explicit `updateStrategy` on StatefulSet with `RollingUpdate` default and `OnDelete` support
- Built-in soft pod anti-affinity (`preferredDuringSchedulingIgnoredDuringExecution`) on `kubernetes.io/hostname` for multi-node clusters, automatically spreading replicas across nodes
- Template-level validation that rejects `maxUnavailable` values exceeding Raft fault tolerance

### Changed

- `pdb.enabled` default changed from `false` to `true`
- `pdb.maxUnavailable` default changed from `1` to `"auto"` (auto-calculate as `floor(replicaCount/2)`)
- `updateStrategy.rollingUpdate.maxUnavailable` added with default `"auto"` (auto-calculate)
- Built-in soft pod anti-affinity is now injected when `affinity` is empty (`{}`)

## 1.0.0 ![AppVersion: 30.1](https://img.shields.io/static/v1?label=AppVersion&message=30.1&color=success&logo=) ![Helm: v3](https://img.shields.io/static/v1?label=Helm&message=v3&color=informational&logo=helm)

**Release date:** 2026-03-19

Initial open-source release.

### Highlights

- Raft-based HA clustering with odd-replica enforcement
- Hardened security context (non-root, read-only rootfs, dropped capabilities)
- Gateway API (HTTPRoute) and Ingress support
- Optional Prometheus metrics sidecar with ServiceMonitor
- Optional External Secrets integration
- PodDisruptionBudget support for quorum protection
- Comprehensive Typesense configuration surface (CORS, analytics, cache, thread pools, logging, health thresholds)
- Topology spread constraints, affinity, and tolerations
