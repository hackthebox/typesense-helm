# Changelog

## 1.1.0 ![AppVersion: 30.1](https://img.shields.io/static/v1?label=AppVersion&message=30.1&color=success&logo=) ![Helm: v3](https://img.shields.io/static/v1?label=Helm&message=v3&color=informational&logo=helm)

**Release date:** 2026-03-19

### Added

- Backup sidecar with restic for scheduled S3 snapshots
- Backup scripts configmap with entrypoint and backup logic
- Pod-0-only backup enforcement
- Configurable cron schedule, retention policy, and restic image
- Extra env/envFrom support for backup credentials

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
