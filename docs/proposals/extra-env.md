# Proposal: `extraEnv` / `extraEnvFrom` on `OdooInstance` and job CRDs

**Status:** draft (not filed upstream). Author: macroscoop integration branch.
**Tracking:** none yet — to be filed once upstream maintainer bandwidth returns.

## Motivation

OCA storage modules (`storage_backend_*`, `fs_storage`, attachments via S3) and
many OCA integration modules (`server_env`, mail-gateway clients, payment
gateway HTTP clients) expect their credentials to come from **environment
variables**, not from `ir.config_parameter` rows in the database. Two reasons
this matters:

1. **Secrets out of the DB.** Putting S3 access keys in `ir.config_parameter`
   means every DB dump (developer copy, staging refresh, accidental
   `pg_dump > /tmp`) carries production credentials. `server_env` exists
   specifically to break that coupling.
2. **Per-environment overrides.** A single Odoo codebase needs to point at
   `prod-attachments` in production and `staging-attachments` in staging
   without DB edits — env vars are the standard pivot.

The current `OdooInstance` CRD has no path to inject environment variables
into the Odoo pods or any of the operator-managed job pods. `spec.configOptions`
exists but is materialised as a plain `ConfigMap` containing rendered
`odoo.conf` (`controller/child_resources.rs::ensure_config_map`), so it
cannot reference Kubernetes `Secret` resources and is the wrong place for
credentials.

Without this, hosted tenants cannot use any OCA module that follows the
"credentials in env, not in DB" pattern. S3-backed attachments is the
nearest concrete blocker, but the surface is broader (SMTP relay creds via
`server_env`, OAuth provider secrets, payment gateway keys, etc.).

## Goals

- Native Kubernetes vocabulary: full `EnvVar` / `EnvFromSource` types so that
  `secretKeyRef`, `secretRef`, `configMapKeyRef`, `configMapRef`, and
  downward-API field refs all work without operator-specific glue.
- One declaration on `OdooInstanceSpec` flows to **every** pod the operator
  spawns for that instance: web Deployment, cron Deployment, and all job
  pods (init, upgrade, backup, restore, staging-refresh, filestore-migrate).
- Per-job CRDs (`OdooInitJob`, `OdooUpgradeJob`, …) may extend or override
  the inherited env for one-shot scenarios (e.g. a manual restore from a
  bucket that the instance itself never reads from).
- Additive; no breaking changes to existing `OdooInstance` manifests.

## Non-goals

- Moving `odoo.conf` itself out of a `ConfigMap` into a `Secret`. The
  current ConfigMap already carries `db_password` and `admin_password`,
  which is its own hardening pass and orthogonal to this proposal.
- Templated env-from-config-options. If a user wants a secret-backed
  `odoo.conf` value, they use `server_env`, which is the OCA-blessed
  pattern and Just Works once env injection exists.
- A reserved-prefix enforcement mechanism beyond a simple validating
  webhook denylist (see "Safety" below).

## Why all job pods, including backup/restore

The shell-only jobs (`backing_up.rs`, `restoring.rs`) don't load the Python
runtime today, so they don't *need* env vars to function. They are wired in
anyway because:

- **Backup completeness once S3 is the attachment backend.** With
  `fs_storage` + S3, the on-disk filestore PVC is largely empty; the
  canonical attachment data lives in the bucket. The current backup job
  dumps `pg` + tars the PVC and produces an incomplete snapshot. The
  planned mitigation is to add an S3-sync sidecar step (e.g.
  `rclone sync s3:src local:/backup/attachments` before the upload step),
  which needs the bucket credentials in the pod. Wiring env now means
  that future step is a pure additive change.
- **Restore symmetry.** Restoring into a fresh bucket prefix or into a
  staging-specific bucket is the natural counterpart and benefits from
  the same plumbing.
- **Mental model.** "Every pod the operator spawns for this instance
  inherits the instance's env" is easier to reason about than a per-job
  carve-out, and matches how the existing `staging_mail_env_vars`
  helper already injects MAIL_* into restore/refresh/neutralize pods.

The cost is ~5 lines per pod-spec call site (∼6 sites), so the
asymmetry-for-purity trade is not worth it.

## API surface

### `OdooInstanceSpec`

```rust
// New fields on src/crd/odoo_instance.rs::OdooInstanceSpec

/// Extra environment variables injected into every pod the operator
/// spawns for this instance: web + cron containers, and the init,
/// upgrade, backup, restore, staging-refresh, and filestore-migrate
/// job pods.
///
/// Use this for non-secret values or for values constructed from
/// `valueFrom.secretKeyRef` / `valueFrom.configMapKeyRef` /
/// `valueFrom.fieldRef`. Merged after the operator's own env, so user
/// entries take precedence except for reserved names (see webhook).
#[serde(default, skip_serializing_if = "Vec::is_empty")]
pub extra_env: Vec<k8s_openapi::api::core::v1::EnvVar>,

/// Extra `envFrom` sources (`secretRef`, `configMapRef`) injected into
/// every pod the operator spawns for this instance. Appended after the
/// operator's own envFrom, so on key collision Kubernetes' last-wins
/// rule applies. Reserved-name keys inside the referenced Secret/CM
/// are not blocked at admission time — see "Safety".
#[serde(default, skip_serializing_if = "Vec::is_empty")]
pub extra_env_from: Vec<k8s_openapi::api::core::v1::EnvFromSource>,
```

The existing CRD already embeds `Affinity`, `Toleration`, and
`ResourceRequirements` from `k8s_openapi`, so the JsonSchema derivation
path is precedent.

### Per-job CRDs

`OdooInitJobSpec`, `OdooUpgradeJobSpec`, `OdooBackupJobSpec`,
`OdooRestoreJobSpec`, `OdooStagingRefreshJobSpec` each grow the same two
optional fields, with a third field controlling merge behaviour:

```rust
#[serde(default, skip_serializing_if = "Vec::is_empty")]
pub extra_env: Vec<k8s_openapi::api::core::v1::EnvVar>,

#[serde(default, skip_serializing_if = "Vec::is_empty")]
pub extra_env_from: Vec<k8s_openapi::api::core::v1::EnvFromSource>,

/// How to combine job-level extra env with the parent instance's.
///   - `Append` (default): instance env first, then job env (job wins
///     on name collision).
///   - `Replace`: ignore the instance env entirely.
/// Useful when a one-shot restore reads from a different bucket than
/// the instance itself.
#[serde(default)]
pub env_inheritance: EnvInheritance,
```

```rust
#[derive(Clone, Debug, Default, Serialize, Deserialize, JsonSchema, PartialEq, Eq)]
pub enum EnvInheritance {
    #[default]
    Append,
    Replace,
}
```

## Implementation sketch

Touch points (Rust):

| File | Change |
|---|---|
| `src/crd/odoo_instance.rs` | Add `extra_env`, `extra_env_from` to `OdooInstanceSpec` |
| `src/crd/odoo_init_job.rs`, `odoo_upgrade_job.rs`, `odoo_backup_job.rs`, `odoo_restore_job.rs`, `odoo_staging_refresh_job.rs` | Add same two fields + `env_inheritance` |
| `src/controller/helpers.rs` | New helper `merge_extra_env(base: &[EnvVar], extra: &[EnvVar]) -> Vec<EnvVar>` with last-wins-by-name semantics; companion `merge_extra_env_from`. |
| `src/controller/child_resources.rs` | Inject merged env into web container (`build_web_deployment`) and cron container (`build_cron_deployment`). |
| `src/controller/states/initializing.rs`, `upgrading.rs`, `backing_up.rs`, `restoring.rs`, `cloning_from_source.rs`, `migrating_filestore.rs` | Inject merged env into each `Container { env, env_from, .. }` constructed for the job pod. Read job-level overrides off the job CRD when present. |
| `src/webhook.rs` | Validating webhook: reject `extra_env` entries whose `name` matches the reserved-prefix denylist (see "Safety"). `extra_env_from` is **not** scanned (would require dereferencing Secrets/CMs at admission time, which we don't want). |
| `src/controller/states/cloning_from_source.rs` | Subtlety: the staging-refresh job clones prod → staging but runs neutralize on the **target** instance. The env it sees must be the **target's** env, not the source's. Already the case if env lives on the target's `OdooInstance` — flag in tests. |

### Merge helper sketch

```rust
/// Merge two EnvVar slices, last-wins by name. Order preserved from
/// `base`, with `extra` overrides applied in-place and new entries
/// appended.
pub fn merge_extra_env(
    base: &[EnvVar],
    extra: &[EnvVar],
) -> Vec<EnvVar> {
    let mut out: Vec<EnvVar> = base.to_vec();
    for e in extra {
        if let Some(slot) = out.iter_mut().find(|x| x.name == e.name) {
            *slot = e.clone();
        } else {
            out.push(e.clone());
        }
    }
    out
}
```

EnvFromSource merging is plain concatenation; Kubernetes already applies
last-wins for keys within an `envFrom` source list.

### Call-site shape

Existing job pod specs look like:

```rust
container: Container {
    env: Some(pg_env),
    // …
}
```

becomes:

```rust
let base_env = pg_env;                                 // operator-owned
let inherited = effective_extra_env(instance, job);    // helper, honours env_inheritance
let env = merge_extra_env(&base_env, &inherited);

container: Container {
    env: Some(env),
    env_from: Some(effective_extra_env_from(instance, job)),
    // …
}
```

For Deployment containers (`build_web_deployment`, `build_cron_deployment`),
the base env is empty today, so the call site is:

```rust
env: Some(merge_extra_env(&[], &instance.spec.extra_env)),
env_from: (!instance.spec.extra_env_from.is_empty())
    .then(|| instance.spec.extra_env_from.clone()),
```

## Safety: reserved env names

The validating webhook rejects user-supplied `extra_env` entries whose
`name` starts with any of these prefixes:

- `PG` — the operator uses `PGDATABASE` etc. to talk to CloudNativePG.
- `ODOO_RC` — Odoo's own config-path env var.
- `BEMADE_` — reserved namespace for future operator-managed env.

`extra_env_from` references (`secretRef`/`configMapRef`) are **not**
inspected at admission time, because doing so would require the webhook
to read referenced Secrets/CMs synchronously. Documented caveat: if a
user mounts a `Secret` via `extra_env_from` that happens to contain a
key called `PGDATABASE`, Kubernetes' last-wins env merging means the
user's value will override the operator's. The mitigation is the
documentation, not the code — same posture as upstream Helm charts that
expose `envFrom`.

## Compatibility

- Existing `OdooInstance` manifests continue to apply unchanged.
- Existing job CRDs continue to apply unchanged.
- CRD schema changes are additive (new optional fields).
- No status-field changes; no state-machine transitions affected.

## Testing

- **Unit (Rust):** `merge_extra_env` with empty/empty, empty/some,
  some/empty, overlap on name, multiple overlaps preserving order.
- **Unit (Rust):** `effective_extra_env` honours `EnvInheritance::Replace`
  vs `Append`.
- **Webhook unit:** reserved-prefix denial for `PGDATABASE`,
  `PGPASSWORD`, `ODOO_RC`, `BEMADE_INTERNAL`.
- **Integration (e2e on test cluster):**
  1. Create a `Secret` with `s3_key` / `s3_secret`.
  2. Apply `OdooInstance` with `extraEnvFrom: [{secretRef: {name: s3-creds}}]`.
  3. Assert env present in web pod, cron pod, init job pod.
  4. Refresh staging from a prod source where prod has different
     `extraEnvFrom`; assert the refresh job pod sees the **target**
     instance's env.
  5. Confirm `OdooBackupJob` pod also has the env (preparation for
     S3-sync sidecar in a follow-up).

## Out-of-scope / follow-ups

- **Backup S3-sync sidecar.** Add an optional sidecar/init step in
  `backing_up.rs` that runs `rclone`/`aws s3 sync` against an
  attachment bucket before the upload step. Lands after this proposal;
  depends on the env wiring done here.
- **Move `odoo.conf` to a Secret.** Separate hardening pass. Removes
  `db_password` / `admin_password` from a ConfigMap.
- **`configOptions` value indirection.** Allow
  `configOptions: { foo: { secretKeyRef: ... } }`. Not needed if users
  adopt `server_env`; revisit if demand emerges.
