# Acknowledgments

This Helm chart would not be possible without the following open-source projects. We are grateful to their maintainers and contributors.

## Typesense

- **Project:** [Typesense](https://typesense.org)
- **Repository:** [typesense/typesense](https://github.com/typesense/typesense)
- **License:** GPL-3.0

Typesense is an open-source, typo-tolerant search engine optimized for instant search experiences. This chart packages Typesense for deployment on Kubernetes but is not officially affiliated with or endorsed by the Typesense project.

## Prometheus Exporter

- **Repository:** [imatefx/typesense-prometheus-exporter](https://github.com/imatefx/typesense-prometheus-exporter)

Provides Prometheus metrics for Typesense server instances. Used as an optional sidecar container when `metrics.enabled` is set to `true`.

## Restic

- **Project:** [Restic](https://restic.net)
- **Repository:** [restic/restic](https://github.com/restic/restic)
- **License:** BSD-2-Clause

A fast, secure, and efficient backup program. Used as the backup sidecar image for scheduled Typesense data snapshots when `backup.enabled` is set to `true`.
