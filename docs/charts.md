# Adding a service

Four things, then three workflows that mostly forward inputs.

```
charts/<service>/
  Chart.yaml              # version is the chart's shape; appVersion is set by CI
  values.yaml             # safe defaults only — never environment-specific
  values.schema.json      # copied from hrbr-dk/ci/schema/
  templates/
    _helpers.tpl
    deployment.yaml
    service.yaml
    ingressroute.yaml
    migration-job.yaml        # {{- if .Values.migration.enabled }}
    preview-postgres.yaml     # {{- if .Values.preview.postgres.enabled }}
deploy/
  dev.yaml
  preview.yaml
.github/workflows/
  ci.yml                  # every PR, no label gate
  deploy.yml              # push to main
  preview.yml             # label-gated, deploy + teardown
```

`deploy/` sits **outside** the chart directory on purpose. `helm package` sweeps up
everything under `charts/<service>/`, so keeping the environment values elsewhere
makes "the same chart with different values" literally true, and keeps the published
artifact free of environment specifics.

## The values contract

Every chart exposes the same top-level keys, whatever the service does. Copy
`schema/values.schema.json` into the chart so `helm lint` enforces it, and extend
under `config` rather than adding top-level keys.

```yaml
image:
  repository: ghcr.io/<org>/<service>
  tag: "" # always set by CI; no floating default
  pullPolicy: IfNotPresent

imagePullSecrets:
  - name: ghcr-pull-secret

replicaCount: 1
env: dev # dev | preview — the only environment discriminator
host: "" # single label + zone

ingress:
  entryPoints: [websecure]
  # No tls key. See below.

service: { port: 80, targetPort: 8080 }
probes:
  path: /health
  port: 8081
  readiness: { initialDelaySeconds: 10, periodSeconds: 10 }
  liveness: { initialDelaySeconds: 30, periodSeconds: 30 }

resources: {}
podSecurityContext: {}
securityContext: {}

database: # backends only
  host: ""
  port: "5432"
  name: ""
  user: ""
  passwordSecret: ""
  passwordKey: password

migration:
  enabled: false
  image: { repository: "", tag: "" }
  backoffLimit: 1
  hookDeletePolicy: before-hook-creation

preview:
  postgres: { enabled: false, image: "", storage: 1Gi }
  seed: { enabled: false, sourceSecret: "", sourceDatabase: "" }

config: {} # free-form app env, rendered as container env vars
```

### Three rules that are not style

**`image.tag` has no default.** Use
`{{ required "image.tag must be set" .Values.image.tag }}`. A `tag: latest` default
means a mistake deploys an arbitrary build instead of failing.

**No `tls:` block on any `IngressRoute`.** Entrypoint-level TLS already supplies a
wildcard certificate for the whole zone. A per-router `certResolver` overrides it and
orders a separate certificate per host, which are then re-issued on every Traefik
restart. The check is:

```bash
kubectl -n kube-system exec deploy/traefik -- cat /data/acme.json | jq '.cloudflare.Certificates | length'
```

Anything but `1` means a router is still carrying a resolver.

**Guard optional blocks as `(.Values.x).y`, never `.Values.x.y`.** This is not
pedantry — it is the exact bug that left one chart unrenderable for weeks:

```
Error: templates/ingressroute-tunnel.yaml:1:14
  executing … at <.Values.ingress.tunnel.enabled>: nil pointer evaluating interface {}.enabled
```

The template only rendered when a values file happened to define `ingress.tunnel`, so
`helm template` passed locally and both deploy paths failed in CI — after the tests
and two image pushes had already run.

## Migrations

One strategy: a Helm `pre-install,pre-upgrade` hook.

```yaml
annotations:
  helm.sh/hook: pre-install,pre-upgrade
  helm.sh/hook-weight: "-5"
  helm.sh/hook-delete-policy: before-hook-creation
```

It is the only approach where a failed migration fails the release without a
hand-written rollback step: with `--atomic`, the running version is untouched and
there is no `kubectl rollout undo` to get wrong.

**The expand/contract requirement.** Every strategy runs the schema change before the
rollout, and every rollback mechanism reverts only the workload. A failed rollout
therefore leaves the *old image* against a *migrated schema*. `--atomic` does not save
you here and reads as though it does.

So: a migration must be backward-compatible with the image it replaces. Add a column,
deploy code that writes both, then drop the old column in a later release. Put this
sentence in the service's README — it is the one thing about this pipeline that will
bite someone who assumes rollback is complete.

## GHCR access — a manual step on every first publish

Chart packages are private, so this recurs once per new chart rather than once ever.

`helm push` of a chart nobody has published before creates the package **private and
unlinked**. Until it is linked, the next run cannot pull it back, and the failure is a
403 on `helm pull` that reads exactly like a login problem.

> Package → Settings → **Manage Actions access** → add the owning repo with **Write**,
> plus every repo that installs it with **Read**.

Cross-repo installs are the ones that catch people, because the grant lives on a
package in a different organisation from the workflow that needs it.

## Adding the workflows

`deploy.yml` and `preview.yml` forward inputs to the reusable workflows; see the
README. `ci.yml` is yours, plus two checks that belong in every repository:

```yaml
- run: helm lint charts/<service> --values deploy/dev.yaml
- run: |
    helm template <service> charts/<service> --values deploy/dev.yaml \
      --set image.tag=ci --set host=ci.example.invalid >/dev/null
    helm template <service> charts/<service> --values deploy/preview.yaml \
      --set image.tag=ci --set host=ci.example.invalid >/dev/null
```

Both values files, not just one. A chart that renders under `dev.yaml` and not under
`preview.yaml` is the normal way this breaks, and it costs seconds on a PR instead of
a full build and two image pushes.
