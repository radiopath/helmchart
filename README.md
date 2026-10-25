# radiopath helm chart

Helm chart for [radiopath](https://github.com/radiopath/radiopath): web pods
(`RADIOPATH_MODE=web`, behind a Service and a Traefik Ingress), worker pods
(`RADIOPATH_MODE=worker`) and a Valkey for the tile cache: three nodes with
Sentinel sidecars (`<release>-valkey-node`) behind HAProxy (`<release>-valkey`),
which always routes to the current master. Database and dataset storage (terrain,
canopy, HF) are external. Chart 2.2 needs radiopath 2.1.0 or later
(`RADIOPATH_MODE`), chart 2.0 and 2.1 radiopath 2.0.0 (the `RADIOPATH_DATASET_*`
variables); for radiopath 1.x use chart 1.x.

`values.yaml` holds the chart defaults; `values-example.yaml` shows the values a
deployment needs. Copy it to `values.local.yaml` for your own release, which
`.gitignore` keeps untracked (as is `values-*.local.yaml`). Secrets
(`DATABASE_URL`, the S3 keys, the SMTP password) come from an existing Secret
named by `envFromSecret`:

```sh
kubectl -n radiopath create secret generic radiopath-env --from-literal=DATABASE_URL=... --from-literal=RADIOPATH_DATASET_S3_ACCESS_KEY=... --from-literal=RADIOPATH_DATASET_S3_SECRET_KEY=...
cp values-example.yaml values.local.yaml   # edit hosts, image repo, env
helm upgrade --install radiopath . -n radiopath -f values.local.yaml --set image.tag=<tag>
kubectl -n radiopath exec -it deploy/radiopath -- /radiopath useradd <name>
```

Registration is off in the chart (`RADIOPATH_REGISTRATION: "false"`); create the
first user with `useradd` as above. Health endpoints, memory sizing and shutdown
behaviour of the app are in the
[radiopath operations reference](https://github.com/radiopath/radiopath/blob/main/docs/reference/operations.md#kubernetes).

## Workers

Worker pods compute everything that needs the datasets: one coverage job each
(a computation uses all CPUs up to the pod limit) and, next to it, link
analyses and HF predictions and maps, one short job per CPU. They download the
HF data set (131 MB) from the bucket into their `/tmp` (an `emptyDir`, the root
filesystem is read-only) at startup. They can be autoscaled on CPU (HPA, needs
metrics-server) and are spread over nodes. Web pods compute nothing and read no
datasets; the dataset variables under `env` are ignored there. `worker.serveWeb`
runs the worker pods in `RADIOPATH_MODE=all` and routes web traffic to them too,
which is fine for a small setup and slow while a coverage is computing.

## Ingress

`ingress.hosts` lists every name a release answers on. Setting
`ingress.canonicalHost` to one of them makes that host the only one serving the
app: it gets its own Ingress, and all the others share a second Ingress with a
Traefik `Middleware` that answers 301, one URL per site, which is what search
engines want. Two Ingresses because Traefik applies the middleware annotation to
every router of an Ingress, so the canonical host has to stand apart.

Certificates come from cert-manager. `ingress.tls.clusterIssuer` defaults to
`letsencrypt`; the chart writes the `cert-manager.io/cluster-issuer` annotation
and derives the Secret names (`<release>-tls`, and `<release>-redirect-tls` for
the redirect Ingress, since one shared name would leave two Certificates fighting
over one Secret with different SAN lists). Issuance is http-01, so every host in
`ingress.hosts` needs DNS pointing at the cluster before its certificate can be
issued. On a cluster without cert-manager set `ingress.tls.clusterIssuer: ""`,
otherwise Traefik answers with its self-signed default certificate.

## Metrics

The chart puts `containerPort: 9090` on web and worker pods and, with
`metrics.podMonitor.enabled`, adds a PodMonitor that selects both components
directly. There is no Service for the metrics port on purpose: a Service
selector is equality only and could not say "web or worker", and the existing
one skips worker pods in production anyway (`worker.serveWeb: false`). The
PodMonitor needs the Prometheus operator CRD; if your Prometheus filters
PodMonitors by label, set `metrics.podMonitor.labels`. The metrics themselves
are described in the [radiopath metrics reference](https://github.com/radiopath/radiopath/blob/main/docs/reference/metrics-and-logging.md#metrics).
