# radiopath helm chart

Helm chart for [radiopath](https://github.com/radiopath/radiopath): web pods
(`RADIOPATH_WORKERS=0`, behind a Service and a Traefik Ingress), worker pods
(`RADIOPATH_WORKERS=1`) and a Valkey for the tile cache. Database and DEM
storage are external.

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

See the [radiopath README](https://github.com/radiopath/radiopath#kubernetes)
for what the values mean.
