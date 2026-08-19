# hrbr-dk/ci

Reusable workflows and composite actions for every service in the hrbr and Tempo
estates. A service repository's three workflows are each under 40 lines because the
work lives here and in the service's own chart.

## Why this repository is public

GitHub only lets a **private** repository's reusable workflows and composite actions
be called from within the same organisation, or the same enterprise — and two free
organisations are not an enterprise. `tempo-dk/backend` cannot call an `hrbr-dk`
private workflow. So this repository is public.

What that exposes is the *shape* of the pipeline. Keep it that way as a rule, not a
habit:

> **Nothing in this repository may name a host, a namespace, or an account.**
> Everything environment-specific arrives as a workflow input or a secret.

A pull request that hardcodes `thygesteffensen.dk`, `hrbrdk`, or a registry account is
wrong even when it works.

## What is here

| Path                                  | What it does                                                      |
| ------------------------------------- | ----------------------------------------------------------------- |
| `.github/workflows/service-deploy.yml`  | push-to-main: build images, publish the chart, install that version |
| `.github/workflows/service-preview.yml` | per-PR environments, and their teardown                            |
| `actions/helm-release/`                 | package → push → install by pinned version                        |
| `actions/preview-namespace/`            | create, label, copy the pull secret                               |
| `schema/values.schema.json`             | the values shape every chart conforms to                          |
| `docs/charts.md`                        | the contract for adding a service                                 |

## Using it

```yaml
# .github/workflows/deploy.yml in a service repository
name: Deploy
on:
  push:
    branches: [main]
concurrency:
  group: deploy-main
  cancel-in-progress: false

jobs:
  deploy:
    uses: hrbr-dk/ci/.github/workflows/service-deploy.yml@v1
    with:
      service: tempo-backend
      namespace: tempo
      host: tempo-api-dev.thygesteffensen.dk
      chart-registry: oci://ghcr.io/tempo-dk/charts
      owner: tempo-dk
      images: >-
        [{"repository":"ghcr.io/tempo-dk/tempo-backend",
          "dockerfile":"src/Tempo.ApiService/Dockerfile","target":"final"}]
    secrets:
      app-id: ${{ secrets.WORKFLOW_APP_ID }}
      private-key: ${{ secrets.WORKFLOW_PRIVATE_KEY }}
```

Pin to `@v1`. The tag moves only for backwards-compatible changes; anything that
changes an input's meaning gets `v2`.

The `app-id` / `private-key` secrets are **optional**. They exist for repositories with
private submodules, where the default `GITHUB_TOKEN` cannot clone a sibling repo.
A service without submodules can omit them entirely and checkout falls back to
`GITHUB_TOKEN`, which can always read its own repository.

## Conventions these workflows assume

- **The chart is the deployable.** Nothing is installed from a local path. Installing
  from the registry is the only way to be sure the artifact that was published is the
  artifact that runs, and it is what lets one repository install another's chart.
- **Versions are immutable.** `<base>-main.<sha7>` and `<base>-pr<N>.<sha7>`; the
  image tag is `<sha7>` or `pr-<N>-<sha7>`. No floating tags are pushed — a `latest`
  or `pr-<N>` tag plus the default `IfNotPresent` pull policy silently pins an
  environment to the first build it ever saw, and the symptom is a deploy that reports
  success while running old code.
- **A preview namespace is named exactly the first label of its host.** Two
  repositories with the same PR number then cannot touch each other's environment.
- **Deleting the namespace is the whole teardown.** There is no DNS or tunnel state to
  unwind; the infrastructure repo owns one wildcard rule that covers the zone.
- **The pull secret is copied, never minted.** A secret built from `GITHUB_TOKEN`
  expires with the job, so a pod that restarts an hour later cannot pull its own image
  and nothing in the event log says why.
