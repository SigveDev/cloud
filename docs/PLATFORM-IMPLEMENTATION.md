# Hauth and Sloyer implementation status

Implementation update: 10 November 2026. The owner approved the plans on 9 November 2026 with **one self-hosted PostgreSQL instance**, separate service databases/roles, and no managed database requirement.

## Created projects

- `E:\Github\hauth` — private repository [SigveDev/hauth](https://github.com/SigveDev/hauth).
- `E:\Github\sloyer` — private repository [SigveDev/sloyer](https://github.com/SigveDev/sloyer).
- `E:\Github\cluster` — new shared PostgreSQL, platform namespace/ingress/TLS, Argo bootstrap/ApplicationSet, promotion workflow, and scoped runtime RBAC configuration.

Both apps have Vite/React/TypeScript dashboards, a Node/TypeScript backend, Docker build/publication workflows, lockfiles, database migrations, meaningful integration/security tests, and their own operational README.

## Implemented paths

Hauth includes persistent OIDC protocol state, code + PKCE login, discovery/signing, branded account interaction, owner bootstrap, password/TOTP validation, explicit app registration and assignment, production admin-network restrictions, protected sessions, and audit. Client credentials are displayed once and encrypted at rest. The initial registration roles support the Sloyer integration.

Sloyer includes Hauth backend login/callback/sessions, project/role checks, GitHub App integration adapters, repository import, onboarding workflow PR generation, durable build queue, provenance-checked release metadata, immutable GitOps manifest writes with race retries, automatic ApplicationSet discovery, workload/TLS/HTTP readiness checks, runtime log retrieval, cancellation of pending promotion, and Git-based rollback.

The database manifests define one PostgreSQL StatefulSet and retained PVC in `data`, with `hauth_app`/`hauth` and `sloyer_app`/`sloyer`, PUBLIC CONNECT revoked, restricted role privileges, bounded connections, and internal-only network access. Apps reference namespace-local Secrets and never receive the database administrator password. A backup helper encrypts logical exports and uploads them off-VPS; enabling its example CronJob requires a real remote, recovery key, verified backup image, policy, and a restore drill.

## Local preview and verification

Hauth: `http://127.0.0.1:18110`. Sloyer: `http://127.0.0.1:18120`. Development API ports are 18111 and 18121. Both preview servers and backend processes were started for browser testing.

The local setup helper generated ignored development-only keys, a local owner, and Sloyer app registration. The development sign-in is stored privately in `E:\Github\hauth\.data\local-access.txt`, not committed or included in this document. Local testing uses embedded PostgreSQL because Docker/native PostgreSQL were not available in this workspace; production mode rejects that backend. The VPS target is still the single shared PostgreSQL pod, not two embedded databases.

Verified locally: TypeScript/production builds, protocol/storage and authorization tests, Kubernetes Kustomize renders, and the browser flow from Sloyer to Hauth and back to an authenticated Sloyer session. The browser check also found and corrected proxy endpoint/origin and form-redirect policy issues. No live database restore, actual GitHub App-driven release, or VPS workload rollout is claimed by those local results.

## Not yet live or fully implemented

Production needs administrator-provided database credentials, persistent Hauth signing/encryption keys, enrolled owner MFA, approved admin networks, GitHub App installations/keys/webhook secret, private registry read credentials where applicable, and offsite backup configuration. These cannot be inferred from a GitHub connector session. The supplied Kubernetes app deployments start with zero replicas until their real configuration and published digests are ready. No existing app is migrated automatically.

The current UI labels secret values as encrypted drafts; they are not yet delivered to containers, and deployments with drafts are blocked. Additional database creation, generic framework/monorepo detection, changing external domains, continuous log streaming, Hauth invitations/email recovery/passkeys, generic consent/API-scope administration, refresh-token rotation, and full multi-user namespace isolation remain roadmap work. Existing app deployments and identity recovery retain an independent GitHub/Argo administrator path.

Follow [cluster bootstrap instructions](https://github.com/SigveDev/cluster/blob/main/docs/SHARED-POSTGRESQL.md) once those reviewed cluster changes are merged. The local authoritative copy is `E:\Github\cluster\docs\SHARED-POSTGRESQL.md`.

## Published validation and infrastructure review

Both private app repositories have been created and source has been pushed. GitHub Actions has successfully run tests, production builds, Linux Docker smoke checks, and GHCR publication. The cluster bootstrap is proposed in [cluster PR #1](https://github.com/SigveDev/cluster/pull/1), not merged or applied. Its render and real PostgreSQL 18 database-isolation jobs passed; they verified own-database migrations, denied cross-database connections, and denied role/database administration for the service roles.

SSH access to the VPS was verified read-only. A memory check showed about 7.7 GiB total and 5.7 GiB available at that moment, supporting the initial small-instance budget subject to load/storage checks. Privileged Kubernetes commands require interactive sudo, so no live cluster resources were changed. The local preview is not the production service.
