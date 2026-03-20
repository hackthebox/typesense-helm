# typesense

![Version: 1.1.0](https://img.shields.io/badge/Version-1.1.0-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 30.1](https://img.shields.io/badge/AppVersion-30.1-informational?style=flat-square)

[![Artifact Hub](https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/typesense)](https://artifacthub.io/packages/search?repo=typesense)

> **Disclaimer:** This Helm chart is maintained by [Hack The Box](https://github.com/hackthebox) and is **not** officially affiliated with or endorsed by the [Typesense](https://typesense.org) project. It is provided on a best-effort basis with no guarantees. Use at your own risk.

## Overview

A production-ready Helm chart for deploying [Typesense](https://typesense.org), an open-source, typo-tolerant search engine built for instant search. This chart deploys Typesense as a StatefulSet with Raft-based clustering for high availability.

### Features

- **High Availability** -- Raft-based clustering with odd-replica enforcement (3, 5, 7, ...)
- **Security Hardened** -- Non-root container (UID 10000), read-only root filesystem, all capabilities dropped
- **Gateway API & Ingress** -- Supports both Kubernetes Ingress and Gateway API (HTTPRoute)
- **Prometheus Metrics** -- Optional sidecar exporter with ServiceMonitor support
- **External Secrets** -- Optional integration with the External Secrets Operator
- **PodDisruptionBudget** -- Protect Raft quorum during node maintenance
- **Comprehensive Tuning** -- CORS, analytics, cache, thread pools, logging, health thresholds

## TL;DR

```bash
helm install typesense oci://ghcr.io/hackthebox/helm/typesense \
  --namespace typesense --create-namespace \
  --set secrets.optional=false
```

## Prerequisites

- Kubernetes >=1.26.0-0
- Helm 3.x
- A pre-created Kubernetes Secret containing at minimum `TYPESENSE_API_KEY`

## Installing the Chart

### From OCI Registry

```bash
helm install typesense oci://ghcr.io/hackthebox/helm/typesense \
  --namespace typesense --create-namespace \
  --values my-values.yaml
```

### Minimal Example

Create a secret with your Typesense API key:

```bash
kubectl create namespace typesense
kubectl create secret generic typesense-secret \
  --namespace typesense \
  --from-literal=TYPESENSE_API_KEY=$(openssl rand -hex 32)
```

Install with defaults (3-replica HA cluster):

```bash
helm install typesense oci://ghcr.io/hackthebox/helm/typesense \
  --namespace typesense \
  --set secrets.optional=false
```

### Production Example

```yaml
replicaCount: 5

resources:
  requests:
    cpu: 500m
    memory: 2Gi

storage:
  className: gp3
  size: 50Gi

pdb:
  enabled: true
  maxUnavailable: 1

gateway:
  enabled: true
  hosts:
    - search.example.com
  parentRefs:
    default:
      name: my-gateway
      namespace: istio-system

metrics:
  enabled: true
  serviceMonitor:
    enabled: true

secrets:
  optional: false

topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: typesense
```

## Uninstalling the Chart

```bash
helm uninstall typesense --namespace typesense
```

> **Note:** PersistentVolumeClaims created by the StatefulSet are **not** deleted automatically. To remove all data:
> ```bash
> kubectl delete pvc -l app.kubernetes.io/name=typesense -n typesense
> ```

## Configuration

### Secret Management

Typesense requires at minimum a `TYPESENSE_API_KEY` environment variable. You have two options:

**Option A: Pre-created Secret (default)**

Create a Kubernetes Secret and the chart will mount it via `envFrom`:

```bash
kubectl create secret generic typesense-secret \
  --namespace typesense \
  --from-literal=TYPESENSE_API_KEY=your-api-key-here
```

**Option B: External Secrets Operator**

If you use the [External Secrets Operator](https://external-secrets.io), enable the ExternalSecret resource:

```yaml
secrets:
  externalSecret:
    enabled: true
    storeName: my-secret-store
    extractKey: path/to/typesense/secret
```

### Exposing Typesense

The chart supports both traditional Ingress and the newer Gateway API:

**Ingress:**

```yaml
ingress:
  enabled: true
  className: nginx
  hosts:
    - search.example.com
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
```

**Gateway API (HTTPRoute):**

```yaml
gateway:
  enabled: true
  hosts:
    - search.example.com
  parentRefs:
    default:
      name: my-gateway
      namespace: istio-system
```

> **Warning:** Enabling both `ingress` and `gateway` simultaneously will create duplicate routing resources. Use one or the other unless you have a specific reason to use both.

### Monitoring

Enable the Prometheus metrics sidecar and optionally create a ServiceMonitor:

```yaml
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
    labels:
      release: prometheus
```

The exporter serves metrics on port 8888 using [typesense-prometheus-exporter](https://github.com/imatefx/typesense-prometheus-exporter).

### Raft Clustering

Typesense uses the Raft consensus protocol. Key considerations:

- `replicaCount` must be **odd** (1, 3, 5, 7, ...) for quorum. The chart rejects even values > 1.
- A 3-node cluster tolerates 1 failure. A 5-node cluster tolerates 2 failures.
- Enable `pdb.enabled=true` to protect quorum during planned node maintenance.
- Nodes discover each other via a headless Service and DNS.

### Persistence

Data is stored on PersistentVolumeClaims attached to each StatefulSet pod:

```yaml
storage:
  className: gp3     # omit to use the cluster default StorageClass
  size: 50Gi
```

> **Warning:** Reducing `storage.size` after initial deployment has no effect. PVC resize depends on your StorageClass supporting volume expansion.

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| affinity | string | `nil` | Affinity rules for pod scheduling. When unset and replicaCount > 1, a soft pod anti-affinity on kubernetes.io/hostname is automatically applied. Set to a non-empty affinity object to override. |
| extraArgs | list | `[]` | Extra command-line arguments for Typesense server (e.g., ["--filter-by-max-ops=200"]) |
| extraEnv | list | `[]` | Extra environment variables for the Typesense container |
| fullnameOverride | string | `""` | Override the full name of the release (optional) |
| gateway.enabled | bool | `false` | Enable or disable Gateway API for the application |
| gateway.hosts | list | `[]` | List of hostnames the Gateway will route traffic for |
| gateway.parentRefs | object | `{"default":{"name":"","namespace":"istio-system"},"extras":[]}` | Configuration controlling which Gateway(s) this HTTPRoute attaches to. |
| gateway.prefix | string | `"/"` | The URL path prefix for the application |
| image.pullPolicy | string | `"IfNotPresent"` | Image pull policy (Always, IfNotPresent, etc.) |
| image.repository | string | `"typesense/typesense"` | Docker image repository for Typesense |
| image.tag | string | `""` | Overrides the image tag. If left empty, it uses the appVersion from Chart.yaml |
| ingress.annotations | object | `{}` | Extra annotations to add to the Ingress resource |
| ingress.className | string | `"nginx"` | The name of the Ingress class to use (e.g., nginx) |
| ingress.enabled | bool | `false` | Enable or disable Ingress for the application |
| ingress.hosts | list | `[]` | List of hostnames the Ingress will route traffic for |
| ingress.prefix | string | `"/"` | The URL path prefix for the application |
| livenessProbe.failureThreshold | int | `2` | Number of failed liveness checks before restarting the container |
| livenessProbe.httpGet | object | `{"path":"/health","port":"http"}` | HTTP GET path and port to check Typesense health for liveness |
| livenessProbe.periodSeconds | int | `10` | Period (in seconds) to perform the liveness check |
| metrics.enabled | bool | `false` | Enable Prometheus metrics sidecar |
| metrics.image.pullPolicy | string | `"IfNotPresent"` | Image pull policy for metrics exporter |
| metrics.image.repository | string | `"imatefx/typesense-prometheus-exporter"` | Metrics exporter image repository |
| metrics.image.tag | string | `"v0.1.5"` | Metrics exporter image tag |
| metrics.port | int | `8888` | Port for Prometheus metrics endpoint |
| metrics.resources | object | `{}` | Resource requests and limits for the metrics sidecar |
| metrics.serviceMonitor.enabled | bool | `false` | Enable ServiceMonitor for Prometheus scraping |
| metrics.serviceMonitor.interval | string | `"30s"` | Scrape interval |
| metrics.serviceMonitor.labels | object | `{}` | Additional labels for ServiceMonitor |
| nameOverride | string | `""` | Override the name of the release (optional) |
| nodeSelector | object | `{}` | Node selector to schedule pods on specific nodes (optional) |
| pdb.enabled | bool | `true` | Enable PodDisruptionBudget for Typesense StatefulSet. Automatically skipped when replicaCount is 1. |
| pdb.maxUnavailable | string | `nil` | Maximum number of pods that can be unavailable during disruption. Auto-calculated as floor(replicaCount/2) when unset, preserving Raft quorum. Can be overridden to a lower value for more conservative disruption handling. Must not exceed floor(replicaCount/2). |
| podAnnotations | object | `{}` | Additional annotations to add to the Typesense pod(s) |
| podLabels | object | `{}` | Additional labels to add to the Typesense pod(s) |
| podSecurityContext.fsGroup | int | `2000` | Group ID for the filesystem of the Typesense container |
| podSecurityContext.runAsGroup | int | `3000` | Group ID for running the Typesense process |
| podSecurityContext.runAsNonRoot | bool | `true` | Ensure the container does not run as root |
| podSecurityContext.runAsUser | int | `10000` | User ID for running the Typesense process |
| readinessProbe.failureThreshold | int | `3` | Number of failed readiness checks before marking the pod as unready |
| readinessProbe.httpGet | object | `{"path":"/health","port":"http"}` | HTTP GET path and port to check Typesense readiness |
| readinessProbe.periodSeconds | int | `5` | Period (in seconds) to perform the readiness check |
| replicaCount | int | `3` | Number of replicas for the Typesense deployment |
| resources | object | `{}` | Resource requests and limits for the Typesense container |
| secrets.externalSecret.enabled | bool | `false` | Enable or disable ExternalSecret creation (requires external-secrets operator) |
| secrets.externalSecret.extractKey | string | `""` | The key path to extract secrets from |
| secrets.externalSecret.storeName | string | `""` | The name of the ClusterSecretStore or SecretStore to use |
| secrets.optional | bool | `true` | Whether the secretRef in envFrom is optional (pods start even if secret is missing) |
| secrets.secretName | string | `""` | Name of the Kubernetes Secret to mount via envFrom. Defaults to <fullname>-secret |
| securityContext.allowPrivilegeEscalation | bool | `false` | Prevent privilege escalation |
| securityContext.capabilities | object | `{"drop":["ALL"]}` | Drop all Linux capabilities |
| securityContext.readOnlyRootFilesystem | bool | `true` | Read-only root filesystem (Typesense only writes to data PVC) |
| service.port | int | `8108` | Port that the Typesense service will listen on |
| service.type | string | `"ClusterIP"` | Kubernetes service type (ClusterIP, NodePort, LoadBalancer) |
| serviceAccount.annotations | object | `{}` | Annotations to add to the ServiceAccount (e.g., for IRSA) |
| serviceAccount.automountServiceAccountToken | bool | `false` | Whether to automount the ServiceAccount token |
| serviceAccount.create | bool | `true` | Whether to create a ServiceAccount |
| serviceAccount.name | string | `""` | Name of the ServiceAccount. Defaults to fullname |
| startupProbe.failureThreshold | int | `10` | Number of failed startup checks before marking the container as unhealthy |
| startupProbe.httpGet | object | `{"path":"/health","port":"http"}` | HTTP GET path and port to check Typesense health for startup |
| startupProbe.periodSeconds | int | `10` | Period (in seconds) to perform the startup check |
| storage.className | string | `nil` | Storage class to use for Persistent Volume Claims (PVC) |
| storage.size | string | `"10Gi"` | Size of the persistent storage volume (e.g., 10Gi) |
| terminationGracePeriodSeconds | int | `300` | Termination grace period in seconds. Typesense recommends 300s to allow graceful shutdown. |
| tolerations | list | `[]` | Tolerations for pod scheduling |
| topologySpreadConstraints | list | `[]` | Topology spread constraints for pod distribution across nodes/zones |
| typesense.analytics.enabled | bool | `false` | Enable aggregated search query analytics |
| typesense.analytics.flushInterval | string | `nil` | How often analytics are persisted to disk in seconds. Unset uses Typesense default (3600) |
| typesense.cache.numEntries | string | `nil` | LRU cache size for search responses. Unset uses Typesense default (1000) |
| typesense.cors.domains | list | `[]` | List of allowed origins (no trailing slashes) |
| typesense.cors.enabled | bool | `false` | Enable CORS for browser/JS access |
| typesense.health.readLag | string | `nil` | Update lag threshold before rejecting reads. Unset uses Typesense default (1000) |
| typesense.health.writeLag | string | `nil` | Update lag threshold before rejecting writes. Unset uses Typesense default (500) |
| typesense.limits.diskUsedMaxPercentage | string | `nil` | Reject writes above this disk usage percentage. Unset uses Typesense default (100) |
| typesense.limits.memoryUsedMaxPercentage | string | `nil` | Reject writes above this memory usage percentage. Unset uses Typesense default (100) |
| typesense.logging.enableSearchLogging | bool | `false` | Log search request payloads |
| typesense.logging.slowRequestsTimeMs | string | `nil` | Threshold in ms for slow request logging (-1 disables). Unset uses Typesense default (-1) |
| typesense.snapshots.intervalSeconds | string | `nil` | Replication log snapshot frequency in seconds. Unset uses Typesense default (3600) |
| typesense.threadPoolSize | string | `nil` | Concurrent request handler threads. Unset uses Typesense default (NUM_CORES * 8) |
| updateStrategy.rollingUpdate.maxUnavailable | string | `nil` | Maximum number of pods that can be unavailable during a rolling update. Auto-calculated as floor(replicaCount/2) when unset, preserving Raft quorum. Can be overridden to a lower value for more conservative rolling updates. Must not exceed floor(replicaCount/2). Ignored when updateStrategy.type is OnDelete. |
| updateStrategy.type | string | `"RollingUpdate"` | StatefulSet update strategy type. Use RollingUpdate (default) for zero-downtime upgrades or OnDelete for manual pod-by-pod control. |

## Upgrading

### To 1.0.0

Initial release. No upgrade considerations.

## Disaster Recovery

See [RESTORE_RUNBOOK.md](RESTORE_RUNBOOK.md) for step-by-step instructions on restoring a Typesense cluster from a restic backup snapshot.

## Contributing

Contributions are welcome. Please open an issue or pull request on [GitHub](https://github.com/hackthebox/typesense-helm).

### Development

```bash
# Install tools
mise install

# Install Helm plugins
mise run setup

# Run tests
mise run test

# Lint
mise run lint

# Generate docs
mise run docs
```

## Acknowledgments

See [ACKNOWLEDGMENTS.md](ACKNOWLEDGMENTS.md) for a list of third-party projects this chart depends on.

## License

This chart is licensed under the [MIT License](LICENSE).

## Maintainers

| Name | Email | Url |
| ---- | ------ | --- |
| Hack The Box SRE | <oss@hackthebox.com> | <https://github.com/hackthebox> |

## Source Code

* <https://github.com/hackthebox/typesense-helm>
* <https://typesense.org>
