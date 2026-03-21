# Stakater Reloader: Never Restart a Pod Again

## The Problem

The Docker registry serves TLS using a cert-manager Certificate with a 24-hour lifetime. cert-manager renews it on schedule. Kubernetes updates the mounted secret on disk. The registry process continues serving the old cert from memory because Docker Registry v2 reads TLS files once at startup and never looks at them again.

Every 24 hours, the cert renews, the registry doesn't notice, and eventually something fails with `x509: certificate has expired`. The fix was to manually restart the pod. I discovered this when the Flux image-reflector-controller couldn't scan the registry because it was presenting a two-day-old cert. Delightful.

This is not a Docker Registry problem specifically. It's a Kubernetes problem generally. Any pod that reads a cert or config at startup and doesn't watch for changes will go stale. Nginx does it. HAProxy does it. Half the things in this cluster probably do it. I just hadn't noticed because most of the certs are consumed by things with their own reload mechanisms (Envoy, Step-CA, etc.).

## Stakater Reloader

[Stakater Reloader](https://github.com/stakater/Reloader) is a Kubernetes controller that watches ConfigMaps and Secrets. When one changes, it finds all Deployments/StatefulSets/DaemonSets that reference it and triggers a rolling restart. It's the "have you tried turning it off and on again" of the cloud-native world, except automated.

The setup:

```
infrastructure/reloader/
├── namespace.yaml
├── repository.yaml    # stakater.github.io/stakater-charts
├── release.yaml       # HelmRelease, watchGlobally: true
└── kustomization.yaml
```

The HelmRelease is minimal:

```yaml
values:
  reloader:
    watchGlobally: true
```

`watchGlobally: true` means Reloader watches all namespaces, not just its own. Without this, it'd only restart pods in the `reloader` namespace, which would be... not useful.

## Opting In

Reloader doesn't restart everything by default — you annotate the deployments you want it to watch:

```yaml
template:
  metadata:
    annotations:
      reloader.stakater.com/auto: "true"
```

The `auto` annotation tells Reloader to watch *all* ConfigMaps and Secrets referenced by the pod — whether mounted as volumes or referenced in `envFrom`/`valueFrom`. For the Docker registry, that covers:

- `registry-tls` — the TLS cert (renewed every 24h by cert-manager)
- `registry-config` — the registry configuration ConfigMap
- `registry-s3-secret` — the SeaweedFS S3 credentials

One annotation, three reload triggers. If I rotate the S3 credentials, the registry picks them up. If I change the config, it picks it up. If the cert renews, it picks it up. No `kubectl rollout restart`, no CronJobs, no prayer.

## The Deploy

Pushed, reconciled, and Reloader came up in under a minute:

```
reloader-reloader-reloader-67c49d44b6-8zm5x   1/1     Running   0
```

Yes, the pod name is `reloader-reloader-reloader`. The Helm release is named `reloader`, the chart is named `reloader`, and the deployment inside the chart is named `reloader`. It's the Forgejo service name problem all over again, but worse.

The registry pod was already restarted by Reloader within seconds of the annotated deployment being applied — it detected the annotation, checked the referenced secrets, and decided a restart was warranted (probably because the secret had been updated more recently than the pod started). New pod came up with the current cert. Old pod terminated.

## What Else Gets This

Anything with cert-manager certificates that doesn't handle reload natively. Right now that's just the Docker registry. But the annotation is there for whenever the next thing needs it. Two lines of YAML and the problem disappears forever.

I should probably go through the cluster and audit which other deployments mount cert-manager secrets. That's a future-me problem. Present-me is just happy the registry won't silently serve expired certs anymore.
