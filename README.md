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
    identity.yaml       # platformIdentity: {appName, type, gitopsRepoUrl, appRepoUrl,
                         # githubOwner, catalogNamespace} - one per app, shared across
                         # every env this app has on THIS cluster. Read by the
                         # tenant-appprojects ApplicationSet to build that app's
                         # AppProject (only .platformIdentity.appName/.gitopsRepoUrl
                         # actually consumed today - no lower-env tier, no CICD
                         # onboarding on this cluster, unlike kind-dev). Same nested
                         # shape as gitops-cluster-dev-tenants's own tenants/<app>/
                         # identity.yaml - one schema for this file across every
                         # cluster, even where a given cluster's own chart doesn't
                         # read every field yet. gitopsRepoUrl/appRepoUrl are bare -
                         # no `.git` suffix, same system-wide convention (see
                         # "History" below).
    <env>/
      identity.yaml      # appName, gitopsRepoUrl, githubOwner, env - same
                         # self-contained shape gitops-cluster-dev-tenants uses, and a
                         # completely different file from the one directly above
                         # despite the identical filename (see that repo's own README
                         # for the two-different-things-same-name gotcha).
```

**Both files are written by `ApplicationEnvironment`'s Composition**
(`idp-service-catalog`, `compositions/applicationenvironment/`) — unlike
`gitops-cluster-dev-tenants`, `tenants/<app>/identity.yaml` here is *not* written by
`NodeJSApplication` (that XRD only ever writes its own `devCluster`'s copy). Every
`ApplicationEnvironment` targeting this cluster for a given app writes it
unconditionally and idempotently, with `managementPolicies` excluding `"Delete"`
(not `spec.deletionPolicy: Orphan` — `provider-upjet-github`'s `RepositoryFile` CRD
has no such field, confirmed live via a real `ReconcileError`; this provider version
uses the newer `ManagementPolicies` mechanism exclusively) — deleting one env's
`ApplicationEnvironment` XR must never delete this cluster-shared file out from
under a sibling env still using it. Cleaning it up when an app is fully
decommissioned from this cluster is a deliberate, separate cluster-admin action, not
automatic — see `idp/docs/service-catalog-design.md` §0 for the full reasoning.

### History: `app.yaml` renamed to `identity.yaml` (2026-08-18)

Until 2026-08-18, the per-app, per-cluster file here was named `app.yaml` (flat:
`appName`/`gitopsRepoUrl`/`githubOwner`, `.git`-suffixed `gitopsRepoUrl`, no
`appRepoUrl` — the only consumer at the time, this cluster's own `tenant-appprojects`
ApplicationSet, never needed it). Renamed and reshaped to match the nested
`platformIdentity:` schema `gitops-cluster-dev-tenants`'s own `tenants/<app>/
identity.yaml` already used (see that repo's own README, "History" section, for the
full cross-repo consolidation this was part of) — one shape for this file
system-wide, even on a cluster where no CICD-onboarding ApplicationSet reads it.
`appRepoUrl` is now populated for real rather than omitted, and `gitopsRepoUrl` lost
its `.git` suffix, matching the one convention every copy of these URLs uses now.
Pre-existing `app.yaml` files were NOT auto-deleted by this rename — the writer's
`managementPolicies` deliberately exclude `"Delete"` (see above) — cleaned up by hand
as a one-time follow-up once the new `identity.yaml` landed for each real tenant.

## Status

Bootstrapped 2026-08-15 alongside `gitops-cluster-kind-prod` and the new cluster
registry (`gitops-cluster-dev/00-bootstrap/cluster-registry/`) — the first real
upper-env cluster in this fleet. Live-verified the same day with a throwaway app:
both `app.yaml` and `<env>/identity.yaml` committed with real, correct content, this
repo's own `tenant-appprojects`/`tenant-onboarding` `ApplicationSet`s picking both up
unprompted, and the orphaned-`app.yaml` teardown behavior (survives the env's own XR
deletion, needs a manual follow-up commit) confirmed exactly as designed.

**`app.yaml` renamed to `identity.yaml` — built and live-verified 2026-08-18**
(`idp-service-catalog` v0.3.21). Forced a reconcile of the real `checkout-api-
kind-prod-prod` `ApplicationEnvironment` XR (annotation bump, since neither its spec
nor the Composition's own resync interval had fired yet on its own) - it wrote a real
`tenants/checkout-api/identity.yaml` here (nested `platformIdentity:`, bare
`gitopsRepoUrl`/`appRepoUrl`) in the same pass that refreshed the pre-existing
`<env>/identity.yaml` to drop ITS `.git` suffix too, both sharing one variable
declaration in `00-cluster-gate.yaml`. Only then pushed this repo's own
`tenant-appprojects` generator switch, specifically to avoid a gap where the
ApplicationSet had nothing matching `tenants/*/identity.yaml` yet - confirmed there
was none: `checkout-api-appproject` was `Synced`/`Healthy` immediately, `AppProject`
`sourceRepos` came out bare as expected
(`https://github.com/jfillman/gitops-checkout-api`), and the real deployment
`Application` (`checkout-api-prod`, still pointed at `idp-service-catalog.git` with
its `.git` suffix intact) stayed `Synced` throughout — the cross-cluster proof that
ArgoCD tolerates a bare `sourceRepos`/`AppProject` boundary against sources that
still carry `.git` elsewhere. The old `tenants/checkout-api/app.yaml` was NOT
auto-deleted (this writer's `managementPolicies` deliberately exclude `"Delete"`, as
documented above) - removed by hand as the one manual follow-up this consolidation
needed. Three older throwaway entries (`deadlock-repro`, `env-pattern-verify`,
`usage-guard-verify`) still have only `app.yaml`, no `identity.yaml` - their
generated `Application`s went stale (still serving pre-cutover `valuesObject`s) and
are expected to self-prune once the ApplicationSet controller's own git-generator
poll catches up; left alone since they're pre-existing test debris, not live tenants.
