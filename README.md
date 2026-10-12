# External Secrets Operator (ESO) — bouc.io

[ESO](https://external-secrets.io/) lets clusters carry only `ExternalSecret`
*pointers* while the secret *material* lives in a real backend. The
`ClusterSecretStore` (the backend pointer) differs per environment:

- **GKE (sandbox)** → GCP Secret Manager via Workload Identity (`gcpsm` provider)
- **pik8s (local)** → a Kubernetes Secret (`boucio-local-secrets`, committed into
  `externalsecrets/local/` and applied into the `external-secrets` namespace) read
  via ESO's Kubernetes provider. pik8s is not internet-exposed, so committing the
  local dev Secret in plaintext is acceptable.

Both clusters use one `ClusterSecretStore` named **`boucio-store`**, and each
service's target Secret (`boucio-<svc>-db-secret`) is identical across envs, so
consuming apps are environment-agnostic.

> **The `ExternalSecret` `remoteRef` differs per environment** — the two
> providers address secrets differently:
> - **gcpsm**: `key: <gcp-secret-name>` (value returned directly, no property).
> - **kubernetes**: `key: <source-Secret-name>` + `property: <data-field>`.
>
> So `externalsecrets/local/` uses `key: boucio-local-secrets` +
> `property: boucio-<svc>-db-password`, while `externalsecrets/sandbox/` uses
> `key: boucio-<svc>-db-password`. This is why the per-env dirs are not identical
> (the "one identical manifest" idea does not hold for the Kubernetes provider).
> `secretStoreRef.name` and `target.name` stay identical.

## Secrets managed here

### Application secrets (gating)

These are applied by `{env}-eso-externalsecrets`, which has `wait: true` and which
`{env}-apps` depends on. Applications do not start until these are synced, which is
intentional: they cannot run without them.

The **Sandbox** column says where the source value comes from on sandbox: *generated*
means ESO mints it (see [Generated vs manual secrets](#generated-vs-manual-secrets));
*manual* means you create it with `gcloud`. On local every value is hand-authored.

| Service | Source secret name | Target K8s Secret (ns `default`) | Keys produced | Sandbox |
|---|---|---|---|---|
| agent-api | `boucio-agent-db-password` | `boucio-agent-db-secret` | `password`, `DATABASE_URL` | generated |
| memory-api | `boucio-memory-db-password` | `boucio-memory-db-secret` | `password`, `DATABASE_URL` | generated |
| chatbot-api *(legacy)* | `boucio-chatbot-db-password` | `boucio-chatbot-db-secret` | `password`, `DATABASE_URL` | generated |
| admin-api | `boucio-admin-db-password` | `boucio-admin-db-secret` | `password`, `DATABASE_URL` | generated |
| portal-api | `boucio-portal-db-password` | `boucio-portal-db-secret` | `password`, `DATABASE_URL` | generated |
| admin-api | `boucio-keycloak-admin-password` | `boucio-admin-app-secret` | `KEYCLOAK_ADMIN_PASSWORD` | manual |
| agent-api | `boucio-agent-redis-password` | `boucio-agent-redis-secret` | `redis-password`, `REDIS_URL` | generated |

`password` is the raw synced value that the bundled Bitnami Postgres reads through
`postgresql.auth.existingSecret`. `DATABASE_URL` is assembled by the ExternalSecret's
`spec.target.template` so the password never lands in a ConfigMap; the app reads it
with a plain `secretKeyRef`.

The agent's Redis follows the identical shape one level down: `redis-password` is the
raw value the bundled Bitnami Redis reads through `redis.auth.existingSecret`, and
`REDIS_URL` is templated alongside it. The key name carries a hyphen (it is Bitnami's
default), so the template reads it with `index . "redis-password"` rather than dot
access, which Go templates reject on hyphenated keys.

> **`mergePolicy: Merge` is mandatory on those templates.** Without it the templated
> `DATABASE_URL` / `REDIS_URL` *replaces* the synced `data` instead of joining it,
> `password` / `redis-password` disappears, and the bundled Postgres or Redis loses
> its credential.

### Add-on secrets (non-gating)

Applied by `{env}-eso-infra-externalsecrets`, which is `wait: false` with **nothing
depending on it**. A missing value here degrades one add-on rather than blocking every
application from deploying.

| Source secret name | Target K8s Secret | Namespace | Key | Sandbox |
|---|---|---|---|---|
| `boucio-datadog-api-key` | `datadog-secret` | `datadog` | `api-key` | manual |
| `boucio-datadog-api-key` | `datadog-secret` | `opentelemetry` | `api-key` | manual |
| `boucio-oauth2proxy-redis-password` | `oauth2-proxy-redis-secret` | `oauth2-proxy` | `redis-password` (+ literal `username`) | generated |
| `boucio-admin-encryption-key` | `boucio-admin-encryption-secret` | `default` | `ENCRYPTION_KEY` | manual |

The Datadog key is deliberately fanned out to two namespaces, because Secrets are
namespace-scoped and it is needed by both the Agent and the OTel collector. Setting it
in a single place is the point of routing it through ESO.

> The Keycloak `kiali-client` secret used to be here too, fanned out to `kiali` and
> `grafana-oauth`. It is not an ESO concern any more: Keycloak owns that value, and the
> `keycloak-kiali-sync` post-install hook Job in `infrastructure/keycloak` writes both
> target Secrets directly. ESO could only ever deliver a value a human had already
> pasted in, which failed silently when it was a placeholder.

`boucio-admin-encryption-key` is optional. admin-api reads it with `optional: true`
and falls back to storing provider API keys in plaintext with a warning, which is the
behaviour from before it was wired up. It is non-gating for exactly that reason.

## Layout

```
external-secrets/
├── namespace.yaml                      # external-secrets namespace
├── base.values.yaml / lcl.values.yaml / snbx.values.yaml   # ESO chart values
├── boucio-local-secrets.example.yaml   # TEMPLATE (repo root, never applied)
├── boucio-local-external-credentials.example.yaml  # TEMPLATE for REAL credentials
├── stores/                             # ClusterSecretStore, applied by *-eso-stores
│   ├── local/                          # kubernetes provider + RBAC (reads external-secrets ns)
│   │   ├── eso-local-reader.yaml
│   │   └── clustersecretstore.yaml
│   └── sandbox/                        # gcpsm provider
│       └── clustersecretstore.yaml
├── generators/                         # SANDBOX ONLY, applied by *-eso-generators
│   └── sandbox/                        # see "Generated vs manual secrets"
│       ├── password-generator.yaml     # Password generator (ns external-secrets)
│       └── pushsecret-*.yaml           # 7 PushSecrets -> GCP Secret Manager
└── externalsecrets/
    ├── local/                          # GATING. k8s provider: key=<source Secret> + property=<field>
    │   ├── boucio-local-secrets.yaml   # committed file-store Secret (dev values only)
    │   ├── externalsecret-agent-db.yaml
    │   ├── externalsecret-memory-db.yaml
    │   ├── externalsecret-chatbot-db.yaml
    │   ├── externalsecret-admin-db.yaml
    │   ├── externalsecret-portal-db.yaml
    │   └── externalsecret-admin-keycloak.yaml
    ├── sandbox/                        # GATING. gcpsm: key=<gcp-secret-name>
    │   └── ... same six ...
    └── infra/                          # NON-GATING add-on credentials
        ├── local/
        │   ├── externalsecret-datadog.yaml          # ns datadog
        │   ├── externalsecret-datadog-otel.yaml     # ns opentelemetry
        │   ├── externalsecret-oauth2proxy-redis.yaml # ns oauth2-proxy
        │   └── externalsecret-admin-encryption.yaml # ns default (optional key)
        └── sandbox/                    # same four, gcpsm remoteRefs
```

### The two local source Secrets, and why they are separate

| File | Tracked in Git? | Holds |
|---|---|---|
| `externalsecrets/local/boucio-local-secrets.yaml` | **Yes**, applied by Flux | Throwaway pik8s dev passwords (the five DB passwords, the Keycloak admin password) |
| `boucio-local-external-credentials.yaml` | **No**, gitignored, applied by hand once | Live third-party credentials: Datadog API key, oauth2-proxy Redis password, admin encryption key |

Committing the first file is a deliberate, documented choice: pik8s is not
internet-exposed and those values are disposable. The second file exists because
**this repository is mirrored to a public GitHub remote**, so a live Datadog API key
or an encryption key committed there would be published. Only its `.example.yaml`
template is tracked.

Applying that one file replaces the separate `kubectl create secret` steps the
datadog, opentelemetry and oauth2-proxy READMEs used to document.

> **No `kustomization.yaml` files.** Each directory holds raw manifests that the
> corresponding Flux `Kustomization` applies directly (autodetect), exactly like
> cert-manager's `issuers/` and `certificates/`. The
> `boucio-local-secrets.example.yaml` lives at the repo root — outside every
> Flux-watched path — so Flux never applies the placeholder Secret (mirrors how
> cert-manager keeps its `*.test-crt.yaml` at the root).

> Flux consumes this directory through the Git submodule under
> `fluxcd/fluxcdboucio/clusters/components/external-secrets/`. **Edit here first,
> validate, commit, then bump the submodule** so Flux reconciles the change.

## Delivery model

ESO is installed by Flux as a single HelmRelease (same pattern as `external-dns`):

- Base HelmRelease/HelmRepository/Namespace:
  `fluxcd/fluxcdboucio/clusters/base/infrastructure/fluxcd-external-secrets.yaml`
- Per-env value overrides are delivered as ConfigMaps generated from
  `base.values.yaml` + `lcl.values.yaml`/`snbx.values.yaml` and referenced by the
  per-env HelmRelease patch.
- The `stores/` and `externalsecrets/` directories are applied by dedicated Flux
  Kustomizations that depend on the operator install, the same shape as
  cert-manager's issuers/certificates:

| Kustomization | Path | `wait` | Depended on by |
|---|---|---|---|
| `{env}-eso-stores` | `stores/{env}` | `true` | the three below |
| `sandbox-eso-generators` | `generators/sandbox` | `true` | `sandbox-eso-externalsecrets` (**sandbox only**) |
| `{env}-eso-externalsecrets` | `externalsecrets/{env}` | `true` | **`{env}-apps`** |
| `{env}-eso-infra-externalsecrets` | `externalsecrets/infra/{env}` | `false` | nothing |

`sandbox-eso-generators` exists only on sandbox and sits between the store and the
gating ExternalSecrets: the PushSecrets need the `ClusterSecretStore` before they
can write, and the app DB passwords must exist before they can be read. Local has no
equivalent Kustomization by design.

The non-gating `sandbox-eso-infra-externalsecrets` deliberately does **not** depend on
it, even though the oauth2-proxy Redis password it reads is generated. Coupling them
would mean a generator or IAM problem also blocks the Datadog key, which is exactly
the cascade the gating split exists to prevent. That ExternalSecret simply retries
until its PushSecret has landed the value.

> **Why the split.** `{env}-apps` depends on `{env}-eso-externalsecrets` with
> `wait: true`, so anything placed in that directory becomes a hard precondition for
> **every** application. That is correct for the DB credentials an app cannot start
> without, and wrong for an add-on API key: putting the Datadog key there would mean
> a missing Datadog value blocks the entire platform from deploying. Add-on
> credentials therefore get their own non-gating Kustomization. Check their status
> with `kubectl get externalsecret -A` rather than through Flux readiness.

## Local

Flux reconciles the operator, RBAC, the `ClusterSecretStore`, every `ExternalSecret`
and the committed file-store Secret. There is **one** manual step, and only for real
third-party credentials:

```bash
# ONE TIME: real credentials that must not be committed (see the table above).
cp boucio-local-external-credentials.example.yaml boucio-local-external-credentials.yaml
$EDITOR boucio-local-external-credentials.yaml     # gitignored
kubectl apply -f boucio-local-external-credentials.yaml
```

To change a committed dev value (DB passwords, Keycloak admin password):

```bash
$EDITOR externalsecrets/local/boucio-local-secrets.yaml
git commit -am "local: update boucio secrets" && git push
# ...then bump the external-secrets submodule in fluxcd/fluxcdboucio and reconcile.
```

Each data key is the secret's name and **must** match the GCP Secret Manager secret
name, so the ExternalSecrets line up across clusters.

> **The committed local Secret is plaintext on purpose.** pik8s is not
> internet-exposed and those values are disposable. That reasoning does **not**
> extend to live third-party credentials, which is why they live in the gitignored
> file instead: this repo is mirrored to a public GitHub remote. To commit
> everything instead, switch to SOPS, but then Flux needs the decryption key.

## Generated vs manual secrets

On **sandbox**, the values no human ever reads back are not typed in at all. A
`Password` generator mints them and a `PushSecret` writes them into GCP Secret
Manager once. Everything downstream is unchanged: the `ExternalSecret`s still read
GCP Secret Manager exactly as before.

```
Password generator ──► PushSecret ──► GCP Secret Manager ──► ExternalSecret ──► Secret
   generators/sandbox/                (source of truth)      externalsecrets/sandbox/
```

**Generated (7)** — manifests in `generators/sandbox/`:

| GCP secret | Used by |
| --- | --- |
| `boucio-{agent,memory,chatbot,admin,portal}-db-password` | bundled Postgres (`auth.existingSecret`) and the app's `DATABASE_URL` |
| `boucio-agent-redis-password` | bundled Redis and the app's `REDIS_URL` |
| `boucio-oauth2proxy-redis-password` | oauth2-proxy sessions and its bundled Redis |

**Not generated.** Everything else in the sandbox `gcloud` block is still created by
hand exactly as before. Two of those are worth calling out, because at a glance they
look like they should have been generated:

| GCP secret | Why not generated |
| --- | --- |
| `boucio-keycloak-admin-password` | Keycloak sets its own admin password from `auth.adminPassword` in its chart values. This secret is only the copy admin-api reads, so a generated value would not match. Becomes generatable once Keycloak reads `auth.existingSecret`. |
| `boucio-datadog-api-key` | Issued by Datadog. Nothing to generate. |

> **The Keycloak password exists in two places and you must keep them equal.**
> `boucio-keycloak-admin-password` feeds admin-api's `KEYCLOAK_ADMIN_PASSWORD`, while Keycloak's
> own admin user comes from `auth.adminPassword` in `infrastructure/keycloak/snbx.values.yaml`
> (which the pre-release scrub set to `CHANGE_ME`). Nothing enforces that they match, and when they
> do not, admin-api's Keycloak calls fail with 401. **This bit on the first sandbox bootstrap.**
>
> Worse, you cannot fix it from the values file after the fact: `KC_BOOTSTRAP_ADMIN_PASSWORD`
> applies only at Keycloak's **first startup**, after which the admin user lives in Keycloak's own
> Postgres. Editing the chart values later changes nothing. Change it in Keycloak itself, through
> the admin console or `kcadm.sh`, and set it to whatever `boucio-keycloak-admin-password` holds.
>
> Check the two sides agree:
> ```bash
> kubectl get secret boucio-admin-app-secret -n default -o jsonpath='{.data.KEYCLOAK_ADMIN_PASSWORD}' | base64 -d; echo
> ```
> Doing this promptly matters on sandbox for a second reason: the admin console is published at a
> public hostname through the Istio ingress, so a leftover `CHANGE_ME` is internet-facing.

### The two policies that make this safe

Every PushSecret sets both, and neither is optional:

- **`updatePolicy: IfNotExists`** — the GCP secret is created only when absent. The
  generator is stateless and does mint a fresh random value on every refresh, but
  that value is discarded rather than written, so the password in use never
  changes underneath a Postgres PVC. It also means a value you create by hand
  wins: ESO will not overwrite it.
- **`deletionPolicy: None`** — deleting a PushSecret (a rename, a restructure, a
  Flux prune) leaves the GCP secret alone. The GCP copy is the durable one; the
  Kubernetes Secret is a cache that can always be rebuilt from it.

Why not point an `ExternalSecret` straight at the generator: ESO generators keep no
state, so a regenerated value would desynchronize from the password the bundled
Postgres baked into its PVC at first init, and recovery is a manual `ALTER USER`.
Pushing to Secret Manager first gives the value a home that outlives the Secret,
the namespace and the cluster.

### Reading a generated value

Nobody ever sees these typed, so read them from the source of truth:

```bash
gcloud secrets versions access latest --secret=boucio-agent-db-password --project "$PROJECT_ID"
```

### Local deliberately does not use generators

Local has no generator and no `PushSecret`, and that is a decision rather than an
omission. Sandbox can generate because GCP Secret Manager is an external source of
truth that outlives the cluster. Local's only source of truth is the committed
`boucio-local-secrets.yaml`, so a generated value would live in a Kubernetes Secret
and nowhere else — and the moment it was regenerated it would no longer match the
password baked into the local Postgres PVCs. Do not "finish the job" by adding
generators to local.

## Sandbox

The GCP-side project setup is created by hand: the API, the service account, the IAM
roles, and the handful of secrets that cannot be generated. The seven machine-only
secrets are created by ESO itself — see
[Generated vs manual secrets](#generated-vs-manual-secrets). Everything on the
Kubernetes side (`ClusterSecretStore`, the Workload Identity SA annotation, the
`PushSecret`s and the `ExternalSecret`s) stays in Git and is reconciled by Flux.

Run once per project, substituting your project for `GCP_PROJECT_ID`
(see [Environment variables](#environment-variables)):

```bash
PROJECT_ID="<your-gcp-project-id>"

# 1. Enable the API
gcloud services enable secretmanager.googleapis.com --project "$PROJECT_ID"

# 2a. The five DB passwords, the agent Redis password and the oauth2-proxy Redis
#     password are NOT created here, and they are not missing either. ESO creates
#     them in this project on its first reconcile, from the PushSecrets in
#     generators/sandbox/: generated once, never rotated, never overwritten.
#     Do not add them back by hand.
#     See "Generated vs manual secrets" above.

# 2b. Application secrets (gating: apps wait for these)
# Manual: Keycloak sets its own admin password from its chart values
# (auth.adminPassword), so this GCP secret is only the copy admin-api reads.
# Generating it would hand admin-api a password Keycloak does not have.
printf %s "<keycloak-admin-password>" | \
  gcloud secrets create boucio-keycloak-admin-password --data-file=- --project "$PROJECT_ID"

# 2c. Add-on credentials (non-gating)
# Manual: issued by Datadog, so there is nothing to generate.
printf %s "<datadog-api-key>" | \
  gcloud secrets create boucio-datadog-api-key --data-file=- --project "$PROJECT_ID"
# Optional. Omit it and admin-api stores provider API keys in plaintext with a warning.
openssl rand -hex 32 | tr -d '\n' | \
  gcloud secrets create boucio-admin-encryption-key --data-file=- --project "$PROJECT_ID"

# later rotations, e.g.:
# printf %s "new-value" | gcloud secrets versions add boucio-datadog-api-key --data-file=- --project "$PROJECT_ID"
# This also works on a GENERATED secret: updatePolicy IfNotExists means ESO never
# overwrites what is already there, so a hand-added version wins. Rotating a DB
# password still needs a PVC re-init or a manual ALTER USER, whatever created it.

# 3. Create the Google service account ESO will impersonate
gcloud iam service-accounts create eso-gke \
  --display-name="ESO GKE secret reader" --project "$PROJECT_ID"

# 4. Grant it read access to Secret Manager
gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:eso-gke@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# 4b. Grant it WRITE access for the generated secrets (the PushSecrets in
#     generators/sandbox/). Deliberately a custom least-privilege role rather
#     than roles/secretmanager.admin: ESO needs to create secrets and add
#     versions, never to delete or destroy them.
#     RUN THIS BEFORE flux bootstrap. Skip it and every PushSecret fails with
#     PermissionDenied on secretmanager.versions.list, which ESO calls FIRST
#     because updatePolicy: IfNotExists has to check whether the secret already
#     exists before it writes. secretAccessor alone cannot list versions.
gcloud iam roles create boucioEsoSecretPusher --project "$PROJECT_ID" \
  --title="ESO secret pusher" \
  --description="Create and add versions to Secret Manager secrets, no delete" \
  --permissions=secretmanager.secrets.create,secretmanager.secrets.get,secretmanager.versions.add,secretmanager.versions.access,secretmanager.versions.list

gcloud projects add-iam-policy-binding "$PROJECT_ID" \
  --member="serviceAccount:eso-gke@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="projects/${PROJECT_ID}/roles/boucioEsoSecretPusher"

# 5. Workload Identity binding: let the ESO controller KSA impersonate the GSA
gcloud iam service-accounts add-iam-policy-binding \
  "eso-gke@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:${PROJECT_ID}.svc.id.goog[external-secrets/external-secrets]"
```

The Kubernetes half of Workload Identity — annotating the ESO controller
ServiceAccount — is delivered automatically via `snbx.values.yaml`
(`serviceAccount.annotations."iam.gke.io/gcp-service-account"`). **No manual
`kubectl annotate` step is needed.**

> Per-store auth (`auth.workloadIdentity.serviceAccountRef`) is the alternative
> to controller-level WI, but a `ClusterSecretStore` has no single owning
> namespace SA, so we annotate the controller instead.

## Environment variables

| Variable | Where | Purpose |
|---|---|---|
| `GCP_PROJECT_ID` | `cluster-vars` ConfigMap (Flux `postBuild.substituteFrom`) | GCP project hosting Secret Manager + GKE. Substituted into `stores/sandbox/clustersecretstore.yaml` (`projectID`) and `snbx.values.yaml` (the WI SA annotation). |

`GCP_PROJECT_ID` is documented in the FluxCD repo README
(`fluxcd/fluxcdboucio/README.md`). If it is unset, Flux substitution leaves the
field blank and the sandbox store will fail to authenticate.

> **The two substitution sites fail differently, so check both.**
> `eso-stores-kustomization.yaml` declares `substituteFrom: cluster-vars` as
> **non-optional**, so a missing ConfigMap fails that Kustomization loudly. The WI
> service-account annotation in `snbx.values.yaml` is substituted by
> `config-kustomization.yaml`, where `cluster-vars` is `optional: true`. A
> ConfigMap that exists but lacks the `GCP_PROJECT_ID` key therefore renders the
> annotation as `eso-gke@.iam.gserviceaccount.com` **silently**, and the only symptom
> is authentication failures at sync time. Confirm with:
>
> ```bash
> kubectl get sa external-secrets -n external-secrets -o jsonpath='{.metadata.annotations}'
> ```

## Verification

```bash
echo "== 1. Flux kustomizations ==" && \
kubectl get kustomization -n flux-system | grep eso; \
echo "== 2. ExternalSecrets, ALL namespaces (want every one SecretSynced/True) ==" && \
kubectl get externalsecret -A; \
echo "== 3. ClusterSecretStore ==" && \
kubectl get clustersecretstore boucio-store; \
echo "== 3b. PushSecrets (SANDBOX ONLY; empty on local is correct) ==" && \
kubectl get pushsecret -n external-secrets; \
echo "== 4. Source keys, committed dev store ==" && \
kubectl get secret boucio-local-secrets -n external-secrets -o go-template='{{range $k,$v := .data}}{{$k}}{{"\n"}}{{end}}'; \
echo "== 5. Source keys, gitignored real-credential store ==" && \
kubectl get secret boucio-local-external-credentials -n external-secrets -o go-template='{{range $k,$v := .data}}{{$k}}{{"\n"}}{{end}}'; \
echo "== 6. Each DB secret must carry BOTH password and DATABASE_URL ==" && \
for s in agent memory chatbot admin portal; do \
  printf "%-8s " "$s"; \
  kubectl get secret "boucio-${s}-db-secret" -n default -o go-template='{{range $k,$v := .data}}{{$k}} {{end}}{{"\n"}}' 2>/dev/null || echo MISSING; \
done
```

Step 6 is the one that catches a dropped `mergePolicy: Merge`: if `password` is gone
and only `DATABASE_URL` remains, the template replaced the synced data instead of
merging with it, and the bundled Postgres has lost its credential.

To confirm the DSN itself resolved (this prints a password, so redirect it if you
are sharing a terminal):

```bash
kubectl get secret boucio-agent-db-secret -n default -o jsonpath='{.data.DATABASE_URL}' | base64 -d; echo
```

### Debugging a secret that will not sync

Start here, one line, every namespace:

```bash
kubectl get externalsecret -A
```

Anything not `SecretSynced` / `True`, read the reason off the object itself:

```bash
kubectl describe externalsecret <name> -n <namespace>
```

Common causes:

| Symptom | Cause |
|---|---|
| All ExternalSecrets failing | `ClusterSecretStore boucio-store` not Ready — check it first |
| Only the `infra/` ones failing on local | `boucio-local-external-credentials` was never applied (expected if you skipped the add-on credentials) |
| One ExternalSecret failing | The named key is missing from the backend: a `stringData` key on local, a `gcloud secrets create` on sandbox (or, for a generated name, its PushSecret has not synced yet) |
| Sandbox, everything failing | Workload Identity binding or `roles/secretmanager.secretAccessor` missing, or `GCP_PROJECT_ID` absent from `cluster-vars` |
| Sandbox, a PushSecret stuck not-Ready | The `boucioEsoSecretPusher` custom role was never created or bound, or the binding has not propagated yet. `kubectl describe pushsecret <name> -n external-secrets` names the missing permission. Read access alone cannot create secrets |
| A PushSecret still `Errored` right after you fixed the IAM | Probably stale status, not a new failure. A condition's `lastTransitionTime` does **not** move while the resource stays in the same failed state, so it keeps showing the original timestamp. Compare the controller's own clock instead: `kubectl logs -n external-secrets deploy/external-secrets --tail=200 \| grep '"controller":"pushsecret"' \| tail -1` and check its `ts` against when you made the change. Recovery took ~90s in practice; `kubectl rollout restart deployment/external-secrets -n external-secrets` forces it |
| Sandbox, only the generated names missing from Secret Manager | Check `kubectl get pushsecret -n external-secrets`; `gcloud secrets list` will not show them until their PushSecrets are Ready |

## Rotation

- **Sandbox:** `gcloud secrets versions add boucio-<svc>-db-password …`; ESO
  re-syncs on `refreshInterval` (1h).
- **Local:** edit `externalsecrets/local/boucio-local-secrets.yaml`, commit, bump
  the submodule; ESO re-syncs on `refreshInterval`. For the gitignored credential
  file, edit and re-apply it.
- A rotated value reaches the app **only after a pod restart**, because both
  `secretKeyRef` env vars and the old ConfigMap approach are resolved once at pod
  start. [stakater/Reloader](https://github.com/stakater/Reloader) is installed for
  this (`clusters/base/infrastructure/fluxcd-reloader.yaml`) and the five api-charts
  carry `reloader.stakater.com/auto: "true"`, so it happens automatically. Add-on
  charts are not annotated; restart those by hand, e.g.
  `kubectl rollout restart deploy/datadog-agent -n datadog`. Kiali and Grafana are the
  exception: the `keycloak-kiali-sync` Job restarts both of them itself after writing
  their client secret.
- **The bundled Bitnami Postgres does not rotate.** It bakes the password into the
  PVC at first init, so changing the secret updates the app's DSN but not the
  server's credential. Until the Bitnami replacement lands, rotating a DB password
  needs an `ALTER USER` against the running instance or a PVC re-init.

## References

- ESO docs: https://external-secrets.io/latest/
- GCP Secret Manager provider: https://external-secrets.io/latest/provider/google-secrets-manager/
- Kubernetes provider: https://external-secrets.io/latest/provider/kubernetes/

## License

[Elastic License 2.0](./LICENSE) — source-available; not OSI open source.
