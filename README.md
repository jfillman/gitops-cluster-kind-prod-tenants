# gitops-cluster-kind-prod-tenants

Which apps are onboarded to the `kind-prod` cluster — same role as
[`gitops-cluster-dev-tenants`](https://github.com/jfillman/gitops-cluster-dev-tenants)
plays for `kind-dev`, per `idp/docs/gitops-strategy.md` §1.

Read by `gitops-cluster-kind-prod/02-argocd-apps/`'s two `ApplicationSet`s — that's
where the generator logic lives (see that repo's own README).

## Layout

```
tenants/
  <app-name>/
    app.yaml            # appName, gitopsRepoUrl, githubOwner - one per app, shared
                         # across every env this app has on THIS cluster. Read by the
                         # tenant-appprojects ApplicationSet to build that app's
                         # AppProject. No appRepoUrl here - unlike
                         # gitops-cluster-dev-tenants's own app.yaml, that field has
                         # no consumer at any cluster other than the devCluster
                         # (still-stubbed CICD onboarding is dev-cluster-only by
                         # construction).
    <env>/
      identity.yaml      # appName, gitopsRepoUrl, githubOwner, env - same
                         # self-contained shape gitops-cluster-dev-tenants uses.
```

**Both files are written by `ApplicationEnvironment`'s Composition**
(`idp-service-catalog`, `compositions/applicationenvironment/`) — unlike
`gitops-cluster-dev-tenants`, `app.yaml` here is *not* written by
`NodeJSApplication` (that XRD only ever writes its own `devCluster`'s copy). Every
`ApplicationEnvironment` targeting this cluster for a given app writes `app.yaml`
unconditionally and idempotently, with `deletionPolicy: Orphan` — deleting one env's
`ApplicationEnvironment` XR must never delete this cluster-shared file out from
under a sibling env still using it. Cleaning it up when an app is fully
decommissioned from this cluster is a deliberate, separate cluster-admin action, not
automatic — see `idp/docs/service-catalog-design.md` §0 for the full reasoning.

## Status

Bootstrapped 2026-08-15 alongside `gitops-cluster-kind-prod` and the new cluster
registry (`gitops-cluster-dev/00-bootstrap/cluster-registry/`) — the first real
upper-env cluster in this fleet.
