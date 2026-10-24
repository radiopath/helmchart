# radiopath helm chart

Helm chart for [radiopath](https://github.com/radiopath/radiopath): web pods
(`RADIOPATH_WORKERS=0`, behind a Service and a Traefik Ingress), worker pods
(`RADIOPATH_WORKERS=1`) and a Valkey for the tile cache: three nodes with
Sentinel sidecars (`<release>-valkey-node`) behind HAProxy (`<release>-valkey`),
which always routes to the current master. Database and DEM storage are external.

`values.yaml` holds the chart defaults; `values-example.yaml` shows the values a
deployment needs. Copy it to `values.local.yaml` for your own release, which
`.gitignore` keeps untracked (as is `values-*.local.yaml`). Secrets
(`DATABASE_URL`, the S3 keys, the SMTP password) come from an existing Secret
named by `envFromSecret`:

```sh
kubectl -n radiopath create secret generic radiopath-env --from-literal=DATABASE_URL=... --from-literal=RADIOPATH_DEM_S3_ACCESS_KEY=... --from-literal=RADIOPATH_DEM_S3_SECRET_KEY=...
cp values-example.yaml values.local.yaml   # edit hosts, image repo, env
helm upgrade --install radiopath . -n radiopath -f values.local.yaml --set image.tag=<tag>
kubectl -n radiopath exec -it deploy/radiopath -- /radiopath useradd <name>
```

Registration is off in the chart (`RADIOPATH_REGISTRATION: "false"`); create the
first user with `useradd` as above. Health endpoints, memory sizing and shutdown
behaviour of the app are in the
[radiopath README](https://github.com/radiopath/radiopath#kubernetes).

## Workers

Worker pods run one coverage job each, and a computation uses all CPUs up to
the pod limit. They can be autoscaled on CPU (HPA, needs metrics-server) and
are spread over nodes; `worker.serveWeb` also routes web traffic to worker
pods, which is fine for a small setup and slow while a coverage is computing.

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
are described in the [radiopath README](https://github.com/radiopath/radiopath#metrics).
