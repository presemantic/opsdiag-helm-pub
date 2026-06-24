# opsdiag-cicd-helm-pub

Public Helm charts for OpsDiag.

## Charts

- [`opsdiag-app-connector`](./opsdiag-app-connector) deploys the customer-side OpsDiag app connector and depends only on the Opsolving `common` library chart.

## OCI install

```bash
helm install opsdiag-app-connector \
  oci://europe-west1-docker.pkg.dev/prod-common-cicd/opsdiag-helm-pub/opsdiag-app-connector \
  --version 0.1.8
```

The chart defaults are compatible with OpenShift restricted SCC: it does not pin
`runAsUser`, `runAsGroup`, or `fsGroup`, allowing OpenShift to inject the
namespace-range random UID while keeping non-root and restricted container
security defaults.

The connector chart renders `/app/config.yaml` from `values.config`. Supply
the app `license` and base Control edge `url` at the top level, and put relay
settings under `relay.license` and `relay.url`; the public chart does not ship
license defaults.
