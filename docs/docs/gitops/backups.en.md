# Postgres backups (CNPG)

!!! note "Source of truth"
    `iac.homelab-live-infra` (`aws/cmoreira-dev/us-east-1/s3-cnpg-backups`: bucket, IAM user and SSM
    parameters), `gitops.cnpg` (`kustomize/barman-cloud-plugin`, `kustomize/<database>/`) and
    `gitops.backstage.homelab` (`kustomize/postgres`).

## Architecture

The databases are CloudNativePG `Cluster`s (operator 1.30.1, 1 instance each). Backups use the
**barman-cloud plugin** (v0.15.1, namespace `cnpg-system`), which replaces the deprecated built-in
`barmanObjectStore`. It does two jobs: it **archives WAL continuously** and takes **base backups** to S3.

- **Bucket** `cmoreira-dev-cnpg-backups-eu`, in `eu-west-1` (the backups hold personal data): private,
  versioned, SSE-S3 encryption, TLS only. Old versions expire after 30 days.
- **IAM user** `cnpg-backups`, limited to that bucket. The keys go to SSM at
  `/homelab/cnpg/backups-aws/{access-key,secret-key,region}` (the cluster has no IRSA, so the key is
  static, delivered through `ExternalSecret`).
- **Per database**, in its own namespace: `ExternalSecret` `cnpg-backups-aws`, `ObjectStore`
  `<database>-backups`, `ScheduledBackup` `<database>-daily` and the `plugins` block on the `Cluster`
  (with `isWALArchiver: true`).
- **Retention** is 14 days (`retentionPolicy` on the `ObjectStore`). The S3 layout is
  `s3://cmoreira-dev-cnpg-backups-eu/<database>/<database>/{base,wals}/`.

## Coverage

| Database | Defined in | Schedule (UTC) | State |
|---|---|---|---|
| n8n | `gitops.cnpg` (`kustomize/n8n`) | 03:00 | active; restore tested on 2026-10-03 |
| litellm | `gitops.cnpg` (`kustomize/litellm`) | 03:10 | active |
| backstage | `gitops.backstage.homelab` (`kustomize/postgres`) | 03:20 | active |
| teupadel | `gitops.cnpg` (`kustomize/teupadel`) | 03:30 | active |

The old `cnpg-backstage` cluster was orphaned (the app uses the database in the `backstage`
namespace) and was decommissioned on 2026-10-03.

## Restoring

A restore creates a **new cluster** from the backup; it does not overwrite the existing one. Minimal
example (in a test namespace, with the same `ExternalSecret` and `ObjectStore` as the original
database):

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: n8n-restore
spec:
  instances: 1
  storage:
    size: 5Gi
  bootstrap:
    recovery:
      source: origin
      database: n8ndb
      owner: n8nuser
  externalClusters:
    - name: origin
      plugin:
        name: barman-cloud.cloudnative-pg.io
        parameters:
          barmanObjectName: n8n-backups
          serverName: n8n
```

To recover to a point in time, add `recoveryTarget.targetTime` under `bootstrap.recovery`.

**Test done (n8n, 2026-10-03):** the restored cluster came up and a Job read the database: the owner
user, 275 migrations and the 4 `settings` rows were there. Workflows and credentials were at 0 because
none existed in n8n yet; worth repeating the test once there is real data.

## Pitfalls

- **Enabling the plugin restarts the instance once** (it injects the sidecar). With 1 instance the app
  sees a short outage.
- **`immediate: true` on the `ScheduledBackup` fails** when the first backup fires during that restart
  (`requested plugin is not available`). So the schedule does not use `immediate`; the first base backup
  comes from a one-off `Backup` (`<database>-initial`) applied once the cluster is healthy.
- **The operator fills in `plugins[].enabled: true`**. Declare it in the manifest, otherwise ArgoCD shows
  the `Cluster` as OutOfSync.
- Each namespace needs its own `ExternalSecret` and `ObjectStore`; they do not cross namespaces.

## Known risks

- A single bucket and a single IAM user for all databases, in the same AWS account as the databases.
- No alert when a backup fails; today you check the `Backup` status and the `Cluster`
  `LastBackupSucceeded` condition by hand.
- The n8n backup protects the data, but the saved credentials are only readable with the
  `N8N_ENCRYPTION_KEY`, which lives in SSM. Keep a copy outside AWS.
- Single instance: the RPO is the WAL archiving interval, and the RTO is the time of a manual restore.
