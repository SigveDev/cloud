# Shared self-hosted PostgreSQL strategy

Approved by the owner on 9 November 2026. This supersedes the managed-database recommendation; no managed subscription is required.

## Decision

Run **one PostgreSQL instance on the existing VPS**, as a one-replica StatefulSet in namespace `data`, with a retained persistent volume and internal ClusterIP service. Hauth, Sloyer, and future services have their own databases and restricted login roles inside this instance.

| Service | Database | Login role | Access |
| --- | --- | --- | --- |
| Hauth | `hauth` | `hauth_app` | Hauth database only |
| Sloyer | `sloyer` | `sloyer_app` | Sloyer database only |
| Future app | Validated app name | App-specific role | Its own database only |
| Maintenance | As required | Protected administrator/backup identity | Never supplied to application pods or browsers |

Internal endpoint: `postgresql.data.svc.cluster.local:5432`. No public PostgreSQL hostname, Traefik route, NodePort, or Hostinger database port. Only backend processes connect; public Vite JavaScript never receives a database password.

## Isolation and credentials

- Revoke PUBLIC CONNECT on app databases and grant it only to their intended role. Restrict public-schema privileges. Verify cross-database connection denial rather than relying only on different passwords.
- App roles have no SUPERUSER, CREATEDB, CREATEROLE, replication, or bypass-RLS. Bootstrap app roles own their own database for migrations; separate migration ownership from runtime CRUD privileges as schemas stabilize.
- Store credentials in namespace-local Kubernetes Secrets. Never commit passwords, connection URLs, backup encryption keys, or bootstrap environments. The PostgreSQL administrator password is not mounted into Hauth or Sloyer.
- Use SCRAM-SHA-256, small pools and role connection limits, plus explicit ingress/egress NetworkPolicies. Initially transport is within the trusted single-node cluster; add verified PostgreSQL TLS before expanding to additional/untrusted nodes or off-server traffic.
- Separate databases are logical access boundaries, not separate physical resource, administrator, backup, or outage boundaries.

## Storage and resource budget

Use a persistent volume claim, never `emptyDir`, and exactly one PostgreSQL replica. Start with a 20-GiB claim, 128-MB shared buffers, 4-MB work memory, a 40-connection server limit, and at most five connections per app process. These are adjustable initial settings; measure live cluster headroom before installation. Local-path claims generally do not enforce a real disk quota, so monitor the whole disk, including images, logs, OS, and other volumes.

Record an image digest and explicit major version before live installation. Back up before upgrades; major-version changes need `pg_upgrade` or dump/restore, not just a new image pointed at an old volume. One PostgreSQL pod on one VPS is not HA. No database operator, Redis, or extra monitoring stack is required just to bootstrap this instance.

## Bootstrap

1. Through the cluster repository/Argo administrator path, create namespaces, protected PostgreSQL bootstrap Secret, service, StatefulSet, volume, and network policies.
2. On a fresh empty volume, initialization creates `hauth_app`/`hauth` and `sloyer_app`/`sloyer` using supplied protected passwords, revokes PUBLIC CONNECT, and limits connections. Init scripts do not rerun on existing storage.
3. Create Hauth and Sloyer namespace-local runtime Secrets with only their own connection URL and application keys. Secrets cannot be referenced across namespaces.
4. Run migrations and owner bootstrap, then verify own-database success, cross-database denial, and data persistence across pod restarts.
5. Configure encrypted off-VPS backups and complete a restore drill before accepting important data.

Future Sloyer database creation must use a separately constrained provisioning process with validated names, fixed operations, quotas, audit, and protected deletion. Its ordinary runtime role does not receive CREATEDB/CREATEROLE. Reserve Hauth/Sloyer databases from app lifecycle deletion.

## Backup and recovery

Schedule `pg_dump` exports for every database plus protected role/grant recovery material, encrypt them, and copy them outside the VPS with a dedicated identity. Keep backup decryption keys outside the server, set retention/failure alerts, and restore into a fresh test instance. A file on the same disk is insufficient.

Initially recover to the last successful logical dump. Do not claim point-in-time recovery without implementing and testing WAL archiving. Planning targets are daily exports and a four-hour manual restore drill, to be adjusted after measurements. Hauth signing/encryption keys require their own protected backups.

Pausing an app never removes its database or volume. App, database, role, PVC, and platform removal are separate operations. Avoid automatic pruning of storage. Node loss requires external backups, while an ordinary pod restart reuses the volume.

## Acceptance

- One persistent PostgreSQL pod, internal-only service, working restart persistence.
- Own-database access works; cross-database access and privileged operations fail for app roles.
- No administrator credential in app pods, browser bundles, or Git.
- Bounded pools, connection/memory/disk limits, credential rotation, migrations, and backup failures are tested.
- Both services and Hauth keys can be recovered into a fresh instance.

## Locations and references

Configuration: `E:\Github\cluster\platform\postgresql`. Operations: `E:\Github\cluster\docs\SHARED-POSTGRESQL.md`. Apps: `E:\Github\hauth` and `E:\Github\sloyer`.

[PostgreSQL roles](https://www.postgresql.org/docs/current/user-manag.html), [database administration](https://www.postgresql.org/docs/current/managing-databases.html), [backups](https://www.postgresql.org/docs/current/backup.html), [Hauth plan](HAUTH-PLAN.md), and [Sloyer plan](SLOYER-PLAN.md).
