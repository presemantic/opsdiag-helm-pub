# Opsolving Public Charts Instructions

## Purpose

This repository publishes customer-deployable Opsolving charts. [`opsolving-app/`](./opsolving-app/) is the unified application stack, and [`opsolving-app-connector/`](./opsolving-app-connector/) is the separately installed customer connector. Release CI publishes public OCI charts below `europe-docker.pkg.dev/presemantic/public/charts`; customer-deployable component images live below the sibling `images` path in the same public repository.

## Structure

[`opsolving-app/`](./opsolving-app/) owns API, Agent, Front, Scheduler, Pipelines, MCP Proxy, VCS, mandatory standalone APISIX, and independent `pgsql` and `pgsql-tables` PostgreSQL 18 instances. The embedded databases use the Docker Official `postgres:18` image and direct chart-owned Secret, ServiceAccount, Service, headless Service, StatefulSet, and PVC templates. Either database may be disabled only when the matching explicit external URL is supplied. [`opsolving-app-connector/`](./opsolving-app-connector/) owns the separately released Connector workload and its Secret-backed runtime configuration.

## Chart Design

Both charts follow the Opsolving Bitnami-style contract and use the Opsolving `common` library from `https://opsolving.github.io/charts/`. Keep one readable Kubernetes object per template. Helpers may calculate chart-specific names and scalar values, but must not render complete resources, resources blocks, affinity blocks, arbitrary objects, or copied/mutated value trees. Generic `extraDeploy`, PodDisruptionBudget, and NetworkPolicy resources are intentionally unsupported.

Values are the public parameter reference. Keep `## @section` and adjacent `## @param` documentation complete, preserve structured component roots, and keep component image defaults in the chart. The App suite `appVersion` is release metadata; Connector `appVersion` must equal its default image tag. Dependencies remain pinned by `Chart.lock` and committed vendor archives so linting and packaging do not fetch at release time.

Customer deployment values should carry licenses, credentials, sources, external persistence choices, exposure settings, and genuine environment overrides. Stable public endpoints, runtime timeouts, and process environment belong in chart defaults. Frequently tuned production controls—`replicaCount`, component autoscaling, and resources—must remain explicit in environment values while chart defaults stay runnable as single replicas with autoscaling disabled.

## Application Contract

APISIX is the only application-routing authority. It runs as a standalone data-plane Deployment with etcd, Admin API, Control API, dashboard, ingress controller, CRDs, Gateway API, metrics Service, and stream listeners disabled. Parent-generated `apisix.yaml` owns OAuth discovery, `/mcp/oauth`, `/mcp/proxy`, `/mcp/self`, API, and Front routes; App VCS remains internal and has no APISIX, Ingress, Route, or Istio path.

Runtime configuration containing infrastructure credentials is rendered to component-specific Secrets mounted read-only at `/app/config.yaml`. App Agent access grants use object entries with a normalized tenant `slug` and a maximum `mode` of `ReadOnly` or `ReadWrite`; legacy scalar grants are compatibility-only and mean `ReadOnly`. App VCS config contains only license, App API, and Connector settings: GitLab/Bitbucket access is tenant-managed in encrypted App Resources Credentials and arrives only in request-bound internal calls. Scheduler keeps its existing config-owned sources. App API, migration, and Pipelines must use the same chart-derived main and Tables database URLs. `PROVIDER_IDENTITY_CUTOVER_MIGRATION_ACK`, `FLOW_INSTRUCTION_CUTOVER_MIGRATION_ACK`, and `FLOW_INSTRUCTION_COMPAT_CLEANUP_ACK` default to false and may be enabled only for their separately backed-up, preflighted migrations.

Connector-backed egress is disabled by default for every App component. An environment that installs the standalone Connector must explicitly set each participating component's `config.connector.enabled` value and its environment-specific Relay URL; the chart must not infer Connector usage from nested source or credential fields. Customer-facing defaults use `https://edge.opsolving.ai` for license verification and Connector grants and `https://relay.opsolving.ai` for Connector client transport. The standalone Connector uses the same Opsolving common OpenShift security-context adaptation as the unified App. Its default pod and container contexts contain no fixed user or group IDs, and with `global.compatibility.openshift.adaptSecurityContext: auto`, OpenShift capability detection also removes any fixed `fsGroup`, `runAsUser`, and `runAsGroup` overrides so the namespace's restricted SCC assigns valid IDs. The single-container pod mounts its read-only projected config Secret with mode `0444`, allowing the OpenShift-assigned arbitrary UID to read it without an ownership init container. The chart must preserve the remaining hardened context and must never create, bind, or require a custom SCC.

## Compatibility

The product and Kubernetes resource brand is Opsolving. Historical wire identifiers such as `X-OpsDiag-*`, `OpsDiag-Payload`, persisted export formats, license component IDs, and Go module paths remain unchanged until a separately versioned protocol migration is implemented. Do not rename them as part of visual or deployment rebranding.

The intentional production rebrand uses new `opsolving-*` releases, StatefulSets, and PVCs. The separate Control PostgreSQL installation is not part of this chart and must never be renamed, recreated, migrated, or removed by App chart work.

## Validation

Run `helm lint` and `helm template` for both charts, render the App chart with [`ci-values.yaml`](./opsolving-app/ci-values.yaml), verify selectors against pod labels, and reject rendered PDB or NetworkPolicy objects. Render App with the OpenShift security API capability and reject fixed `runAsUser`, `runAsGroup`, or `fsGroup` values in workload security contexts. Verify exactly two chart-owned PostgreSQL StatefulSets, official `postgres:18`, stable client Services, StatefulSet names without a database-version suffix, and the APISIX standalone configuration. A production name migration may temporarily mount retained legacy PVC names through `primary.persistence.existingClaim`; do not rebind or recreate those data volumes merely to rename them.

Do not run Helm install, upgrade, rollback, or cluster mutations unless the user explicitly authorizes deployment. Before publishing, inspect the diff and status, preserve the current branch, and use the repository's timestamp/hash release-tag convention.
