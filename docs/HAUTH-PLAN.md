# Hauth: identity platform implementation plan

Status: approved by the owner with shared self-hosted PostgreSQL, 9 November 2026. Implementation has started in the independent local Hauth and Sloyer projects; this plan does not assert a live deployment.

Source repository: `SigveDev/hauth`, local checkout `E:\Github\hauth`. Canonical issuer and hosted login: **https://hauth.sigve.cloud**. Administrator console: **https://hauth.sigve.cloud/admin**, separately protected. Deployments and infrastructure remain in `SigveDev/cluster`.

Related plans: [Sloyer](SLOYER-PLAN.md) and [database recommendation](DATABASE-STRATEGY.md).

## 1. Goal and protocol decision

Hauth is the shared identity service for personal sigve.cloud applications. Provide a Microsoft Entra-inspired experience: app registrations, client IDs, redirect URLs, hosted login, users, assignments, API permissions, administrator consent, app roles, credentials, sign-in logs, and session management.

Build an alternative identity provider, **not a new incompatible authentication protocol**. Use OpenID Connect for login and OAuth 2.0 Authorization Code with PKCE for delegated authorization, following current OAuth security best practices. This makes Hauth usable by standard web frameworks and future integrations without requiring a custom browser token format.

Hauth's UI, registration model, policies, and integrations are custom product work. Use a maintained protocol implementation for authorization codes, token issuance, discovery, and token validation rather than implementing cryptography or an authorization server from scratch. Recommended TypeScript protocol engine: `oidc-provider` from panva, subject to a compatibility/security review and a pinned supported version at implementation time. Using a certified engine does not automatically certify the completed Hauth deployment.

Start with one private directory, `sigve-cloud`, invited accounts only. Full Entra replication, multi-tenant identity federation, SAML, SCIM, enterprise conditional access, and public self-service app registration are later possibilities, not initial requirements.

## 2. User-facing experiences

### Hosted account pages

- Branded sign-in with the requesting application's name and verified identity.
- Password login plus MFA initially; add passkeys through a maintained WebAuthn library in a later phase. Administrative accounts require MFA before public deployment use.
- Invitation acceptance, verified email, password reset, recovery-code setup, account profile, active sessions, and sign out.
- Permission/consent screen for approved delegated API scopes, showing what app is asking for what access. First-party administrator-approved apps may skip repeated consent, but cannot skip user assignment.
- Clear denied-access and expired-link pages that do not leak whether an arbitrary account exists.

Do not allow an application's query parameters to supply arbitrary HTML, executable branding, or an unvalidated post-login destination. Application branding is administrator-controlled.

### Administrator console

| Area | Capabilities |
| --- | --- |
| Overview | Login failures, integration health, expiring client credentials, recent security changes |
| App registrations | Name, immutable app ID/client ID, client type, owner, enabled state, approved URLs |
| Authentication | Exact redirect URIs, allowed logout destinations, supported flow, PKCE and session policies |
| Credentials | Generate/expire/rotate confidential client credentials; reveal plaintext only once |
| API registrations | API audience identifiers, exposed delegated scopes, application roles |
| Permissions | Requested permissions, granted consent, assignment requirements, role assignment |
| Users | Invite, verify, disable, reset recovery, revoke sessions, assign applications/roles |
| Sessions | Hauth SSO sessions and refresh grants, last activity, revoke controls |
| Activity | Sign-in events and administrator audit history with outcomes and correlation IDs |
| Branding | Instance defaults and reviewed app display metadata |

Use the same readable 1600px-wide responsive design direction as Sloyer. Do not expose raw tokens, password hashes, private keys, or client secrets in tables, browser telemetry, or logs.

## 3. Application and permission model

| Concept | Hauth meaning |
| --- | --- |
| Directory | The identity boundary containing users, registrations, assignments, and policy; one initially |
| App registration | Protocol configuration and global identity of an application, including its client ID |
| Directory application instance | The local enabled/assigned/consented instance, analogous to a service principal; one-to-one in the first single-directory version |
| Client | Browser SPA/public client, backend web/confidential client, or later machine client; flows differ by type |
| API resource | An intended token audience, such as the Sloyer API, with explicit scopes and app roles |
| Delegated permission | A scope requested by a client on behalf of a signed-in user; bounded by the user's actual authorization |
| Application permission | A machine identity's explicitly granted capability with no human user; later client-credentials feature |
| App role | An application-specific assignment, such as `sloyer.viewer`, enforced by its backend |
| Grant/consent | Record of who authorized a client to request specified permissions, with revocation |

An app registration is not a permission grant. A user account is not permission to deploy. `openid`, `profile`, and `email` identify a user; they do not authorize project changes. Scope consent does not turn a viewer into an administrator. Hauth directory administration is also separate from ownership of a Sloyer project.

Begin with per-application user assignments and directory/application administrator roles. Add groups and group assignment later. Validate every grant against the client, API audience, allowed scope definitions, user assignment, and consent policy. Unknown or disabled scopes are rejected.

Clients are created by authenticated administrators initially. Do not enable anonymous dynamic client registration. Disabling an app prevents new authorizations and token refresh; revocation behavior for already-issued access tokens must be documented.

## 4. Architecture and deployment

Recommended repository structure:

```text
hauth/
  apps/console/                  # Vite + React + TypeScript administration UI
  apps/server/                   # Node/TypeScript identity server and account interactions
  packages/contracts/            # Validated admin API and identity/configuration schemas
  packages/storage/              # PostgreSQL adapters, migrations, transactions, expiry jobs
  packages/ui/                   # Shared branded account components
  tests/protocol/                # Standards and negative integration tests
  tests/browser/                 # Login, consent, administration, recovery
  docs/                         # Registration, integration, operation, recovery
  .github/workflows/             # Checks, build, publication; no cluster-admin credentials
```

Serve the administration UI and hosted interactions from the identity origin. Mount the OIDC engine using a supported Node HTTP integration; select framework details after validating adapter compatibility. Account interactions call the protocol engine's supported interaction API rather than inventing a parallel token endpoint.

Use a dedicated `hauth` Kubernetes namespace, ClusterIP service, Traefik ingress, and cert-manager certificate. Its service account has no deployment-control privileges and no automatic API token mount unless specifically required. Keep protocol signing keys, cookie encryption keys, MFA encryption keys, database credentials, and email credentials in protected platform Secrets managed independently of application source.

Persist users, registration settings, grants, sessions, short-lived protocol artifacts, revocation state, and audit events in **self-hosted PostgreSQL**. Use a production storage adapter with atomic one-time code consumption, TTL cleanup, refresh-family transactions, and restart-safe sessions. In-memory development adapters are not acceptable in deployment. Avoid Redis as a required first dependency; prove PostgreSQL can implement the necessary expiry/rate-limit/session semantics.

Start with one small Hauth server replica and measured CPU/memory limits, since password hashing is deliberately expensive. This is not high availability: shared local storage does not remove the identity server's single-VPS failure point. Later scaling requires shared rate limits, database-backed state, stable issuer/keys, and concurrency tests.

Allow only required network paths: public browser protocol/login endpoints, restricted admin console/API, internal PostgreSQL connections restricted by NetworkPolicy, DNS, and the configured mail service. Standard NetworkPolicy cannot match DNS hostnames; implement an explicit egress strategy and provider allowlisting where supported.

## 5. Protocol contract

The canonical production issuer is `https://hauth.sigve.cloud`. Its exact spelling and path are permanent configuration, not derived from untrusted Host headers. Local/test environments use distinct issuers and signing keys; never publish a localhost issuer for production.

Endpoint paths below are proposed; configure and verify the actual engine routing and publish the resulting URLs through discovery:

| Endpoint | Purpose |
| --- | --- |
| `/.well-known/openid-configuration` | Authoritative issuer metadata and supported flows/algorithms |
| `/oidc/auth` | Authorization request and hosted interaction entry |
| `/oidc/token` | Code exchange, refresh, and later client credentials |
| `/oidc/jwks` | Public signing keys; never private key material |
| `/oidc/me` | OIDC user information for an appropriate access token |
| `/oidc/token/revocation` | Supported token/grant revocation |
| `/oidc/session/end` | Supported RP-initiated logout with registered destinations |
| `/login` and `/interaction/*` | Branded account and consent interactions |
| `/admin` and `/api/admin/*` | Restricted administrator console and API |

Define a separate authenticated integration contract for assignment freshness, for example a client-authenticated `POST /api/integrations/assignment-check`. It returns only that authenticated client's assignment/role version for an authorized subject and never global directory data. This is a Hauth extension, not an OIDC-standard endpoint. Use a dedicated constrained credential or a specifically designed client-authenticated request; do not accept an ID token as an arbitrary API bearer token. Alternatively implement verified revocation events plus periodic reconciliation, provided the five-minute policy below is met. The private administrator API is not the public relying-party lookup API.

Support Authorization Code + PKCE S256 from the first release. Disable implicit and resource-owner-password grants. Client authentication for confidential apps is separate from PKCE. Public SPAs cannot keep a client secret; future SPA support must enforce PKCE and an exact allowed-origin policy. Prefer a backend-for-frontend session pattern for sensitive apps such as Sloyer and the Hauth console.

Register exact HTTPS callback URLs and exact logout destinations; no wildcard subdomains, arbitrary `returnTo`, or prefix matching. Allow explicit loopback callbacks for separate local-development registrations only. Never redirect errors to an unvalidated redirect URI.

Each client validates authorization response state; OIDC clients validate nonce where used. The server binds codes to the client, redirect URI, PKCE verifier, approved user/grant, and short expiry. Codes are atomically single-use. Refresh tokens rotate with replay-family handling and defined concurrent-refresh behavior.

Use asymmetric signing keys and an explicit approved algorithm list. Token consumers verify signature, issuer, audience, expiry, and the required permissions; reject wrong token type and wrong audience. An ID token is a login assertion for its client, not a general API bearer token. Decide access-token format with the protocol engine; if JWT access tokens are used, define the audience, scopes, role claims, and versioned claim contract and validate them with maintained libraries.

Initial lifetime targets, configurable and to be tested: authorization code about 60 seconds, access/ID tokens about 5–10 minutes, SSO idle timeout about 30 minutes and maximum about 12 hours, refresh grants with an explicit bounded lifetime. These are proposed policy settings rather than universal protocol defaults. Background machine clients never inherit an interactive user's session.

Publish stable `sub` values and use `(iss, sub)` as the application identity key. Do not use mutable email addresses as database user identifiers. Release only claims the client is entitled to receive; `email_verified` must reflect actual verification.

## 6. Sloyer production app registration

Create a first-party registration **Sloyer production**, configured as a confidential web client with a backend-for-frontend. IDs below are logical names, not actual issued identifiers or credentials:

```yaml
displayName: Sloyer production
clientType: confidential-web
issuer: https://hauth.sigve.cloud
clientId: <HAUTH_SLOYER_CLIENT_ID>
grantTypes: [authorization_code, refresh_token]
pkce: S256-required
redirectUris:
  - https://sloyer.sigve.cloud/auth/callback
postLogoutRedirectUris:
  - https://sloyer.sigve.cloud/
requestedScopes: [openid, profile, email]
assignmentRequired: true
appRoles: [sloyer.viewer, sloyer.deployer, sloyer.admin]
```

`refresh_token`/`offline_access` is enabled only if the reviewed Sloyer session design needs it; basic login does not require unlimited offline access. Keep development/staging as separate registrations with exact callback ports/URLs, credentials, assignments, and data.

The Hauth administrator assigns the owner's verified account to Sloyer and grants the appropriate role. A newly invited Hauth user receives no Sloyer privileges by default. Hauth can include the assigned application roles through a documented client-specific claim; it must not leak every application's assignments to every client.

Browser flow:

1. Browser requests Sloyer. Without a valid Sloyer session, its backend starts `/auth/login` and saves short-lived state, nonce, PKCE verifier, and a validated local destination.
2. Browser redirects to Hauth's discovered authorization endpoint with the registered client and callback. Password/MFA entry happens only on Hauth.
3. Hauth authenticates the user, checks Sloyer assignment, and obtains or applies valid consent. It returns a short-lived authorization code to the exact Sloyer callback.
4. Sloyer's backend validates state and exchanges the code over HTTPS with the PKCE verifier and its confidential-client authentication. It validates the resulting ID token and resolves the `(issuer, subject)` user.
5. Sloyer creates an opaque, server-side application session and sends a host-only Secure, HttpOnly cookie. Tokens and the confidential-client secret never enter browser JavaScript or localStorage.
6. Every Sloyer API route enforces session, assigned app role, and project authorization. Mutations also enforce CSRF/origin policy. Login does not mean permission to run deployments.
7. Logout invalidates Sloyer's local session and may initiate Hauth logout using the registered return URL. Do not claim that clearing one cookie signs the user out of every relying application.

Sloyer's background deployment worker uses its own GitHub and infrastructure credentials. It does not store a user's Hauth token as an indefinite deployment credential. Persist who authorized the job and enforce a defined policy for queued work if their account/assignment is later revoked.

Initial queued-work policy: recheck authorization before privileged dispatch and before release promotion. If the actor is disabled, loses access, or the required fresh check cannot complete, hold the job for administrator review rather than promoting. Do not automatically remove a currently healthy serving application solely because its original deployer lost access. Service-triggered auto-deploys have an explicitly assigned project automation identity/policy, not an immortal copy of an owner's interactive login.

## 7. Sloyer authorization contract

| Role | Allowed actions |
| --- | --- |
| `sloyer.viewer` | View authorized project status and deployment metadata; log access only through a separate explicit permission because logs can contain sensitive data |
| `sloyer.deployer` | Viewer capabilities plus deploy/retry/rollback and approved non-sensitive settings for assigned projects |
| `sloyer.admin` | Manage assigned projects, runtime secrets, domains, GitHub connections, database attachment and destructive actions, subject to reauthentication and confirmation |

Hauth supplies identity and coarse app assignments. Sloyer owns project membership and action policy. Never rely on hiding buttons as authorization. Project/log/secret permissions are evaluated server-side on each request and streaming connection.

For the first owner-only deployment, bind the assigned owner subject to the platform owner role. Add and test project isolation before adding other users. Assignment revocation must invalidate application authorization within a defined window: initially no more than five minutes using a cached authoritative assignment/version check, with immediate handling where an authenticated revocation event is implemented. Fail closed for privileged mutations when fresh authorization cannot be established. Existing self-contained JWTs are not instantly revoked merely because a database row changed.

OAuth API scopes for a future CLI/external integration are separate from these UI roles: examples could be `sloyer.deployments.read` and `sloyer.deployments.write`, targeting an explicitly registered Sloyer API audience. Do not mint these before the API validates the access-token contract. Client credentials belong to a later machine-identity phase, with no human impersonation.

## 8. Credentials, user recovery, and session security

- Passwords use Argon2id through a maintained library, with unique salts and a benchmarked work factor based on current guidance. Never encrypt passwords for later recovery. Bound login concurrency to avoid exhausting this small VPS.
- Encrypt TOTP seeds and other secrets that must be recoverable using an application encryption key stored outside the database. Hash recovery codes, reset/invitation tokens, and opaque session handles where their lookup scheme permits. Never place these in telemetry.
- Use short-lived, single-use invitation, verification, and reset links. Store their digest; return generic responses to prevent account enumeration. Configure a real transactional email provider and test delivery before depending on email recovery. Do not run a mail server on the VPS as part of this project.
- Keep confidential-client credentials distinct from user passwords and signing keys. Store only a verifier when the chosen authentication method permits; if the protocol engine requires retrievable client secrets, use envelope encryption with a separately stored key. One-time display, expiry, rotation overlap, and audit are mandatory. `private_key_jwt` can be introduced after the basic confidential-client flow.
- Hauth SSO cookies are host-only, Secure, HttpOnly, and appropriately SameSite; no `Domain=.sigve.cloud` shared login cookie. Sloyer and other applications maintain their own sessions. Choose the callback response mode and SameSite policy together and test cross-site flows.
- Enforce CSRF on account/consent/admin mutations, strict CORS, CSP and safe asset handling, trusted-proxy configuration, login throttling, and bounded requests. Do not enable arbitrary URL fetching from client-provided logos, JWKS URLs, or post-login destinations.
- Rotate signing keys with overlapping public-key publication for the lifetime of existing tokens and consumer caches. Back up private signing/encryption keys securely; persist keys across restarts rather than regenerating them silently.
- Account disablement, role changes, credential reset, and suspected token reuse create audit events and revoke the appropriate sessions/grants. Include a tested mechanism for relying apps to refresh authorization state; instant global logout is a separately implemented feature.

## 9. Storage and recovery model

Recommended first deployment: a dedicated `hauth` database and restricted login role inside the single PostgreSQL StatefulSet shared by all services on the VPS. See [database strategy](DATABASE-STRATEGY.md) for the shared StatefulSet, database/user isolation, backups, and capacity limits.

Suggested entities: directories, users, verified identities, password credentials, MFA credentials, applications, local application instances, client credentials, redirect registrations, API resources, scope definitions, app roles, user assignments, grants, consent, sessions, short-lived protocol artifacts, refresh families, audit events, and migration state.

Use explicit foreign keys, uniqueness constraints, transactions for one-time artifacts and refresh reuse, expiration indexes/cleanup, and tenant/directory keys even in the single-directory schema. Keep protocol adapters aligned with the engine's required semantics; a generic JSON blob cache is not enough if it loses atomic consumption.

Use small connection pools. Verify transaction-pool compatibility of the adapter; migrations, exports, or operations that need session-scoped state use direct connections. Database ownership/migration credentials are separate from runtime credentials. Store connection strings in platform Secrets with an explicitly configured transport policy (internal SCRAM initially; add verified TLS before traffic crosses trusted cluster networking), never in Vite variables or Git.

Define and test recovery of the database, signing keys, encryption keys, issuer settings, and administrator access together. A database backup alone cannot decrypt MFA seeds or preserve signing continuity if key material is lost. Secure recovery exports separately from ordinary application backup access. Start with targets of at most 24 hours data loss for an independently retained export and four hours manual recovery, then tighten based on measured restore tests and the configured backup retention and measured restore capability. These are engineering targets, not availability guarantees.

## 10. Bootstrap and control-plane independence

Avoid a dependency loop where Sloyer must log in through Hauth to deploy or repair Hauth, while Hauth can only be managed through Sloyer.

Bootstrap Hauth through the existing GitHub/Argo administrator path, independently of Sloyer. Its platform definitions, database credentials, and recovery keys are administrator-owned. Register Hauth's own administrator console as a confidential first-party client, but constrain that console/API to private administrator network access initially. Public login/protocol endpoints remain available to registered applications.

Provision the first owner through a one-time, audited bootstrap command or job accessible only to an administrator, not a permanently public setup page. Enroll MFA before enabling privileged actions. Remove/disable bootstrap capability after successful setup.

Maintain an offline documented recovery path through restricted GitHub/Argo/provider access, with encrypted recovery material, that does not depend on Hauth login. Never introduce an anonymous production bypass query parameter or leave a default administrator password. Sloyer does not gain permission to edit Hauth's keys, issuer, registrations, or database through normal application deployment operations.

## 11. Implementation phases

### Phase H0 — Contracts and protocol spike

Choose the supported protocol engine/runtime and PostgreSQL adapter, review threat boundaries, establish exact issuer/client/claims contracts, and prove Authorization Code + PKCE with a minimal standard client. Validate persistence, one-time code consumption, rotation, and transaction-pool compatibility before building the full console.

Deliverable: checked-in protocol decisions, test fixtures, database access plan, and a proven standard client exchange.

### Phase H1 — Private owner identity MVP

Build the branded account interactions, invited owner account, password+MFA flow, persistent sessions, read-only registration diagnostics, and administrator bootstrap/recovery. Configure HTTPS, signing/encryption keys, PostgreSQL, outbound email, and off-VPS backups.

Deliverable: owner can log in after a server restart; missing database/keys cause a controlled failure rather than a new issuer or unlocked admin. Negative login/reset tests pass.

### Phase H2 — App registrations and Sloyer integration

Build registration CRUD, exact redirects/logout URLs, client credentials, assignments and application roles. Create Sloyer's production and development registrations. Implement Sloyer BFF login/callback/logout, session authorization, denied-access pages, role enforcement, and revocation/version checks.

Deliverable: Sloyer redirects to Hauth, the assigned owner returns signed in, an unassigned Hauth account is denied, and viewer/deployer/admin permissions are enforced by the API. This is required before Sloyer's public management launch.

### Phase H3 — Account administration and permission lifecycle

Complete user invitations, email verification/recovery, API scope definitions, consent review/revocation, client credential rotation, sign-in logs, session revocation, key rotation, recovery drills, and step-up authentication for destructive administration.

Deliverable: app disablement, user disablement, credential rotation, and permission removal have tested effects within the documented revocation window. Complete an independent security review before wider use.

### Phase H4 — Reusable developer integration

Publish standard OIDC integration guides and small helpers for Vite+backend applications, example registrations, and Sloyer onboarding configuration. Add passkeys, trusted relying-party logout propagation, and approved machine identities/scopes as separately tested features.

Deliverable: a new app can integrate through discovery and its own registration without sharing Sloyer's credentials. Public-SPA support has explicit limitations and is not the default for deployment administration.

### Phase H5 — Expansion only when needed

Groups, directory federation, multiple tenants, third-party identity providers, enterprise permission workflows, and delegated registration ownership follow real requirements. Do not describe these as complete just because their table names exist.

## 12. Acceptance and security tests

- Standard OIDC client and conformance-oriented tests for discovery, signatures, code exchange, PKCE, client authentication, claims, and logout; run applicable OpenID conformance tests when practical.
- Reject changed callback URLs, reused/expired codes, bad state/nonce, wrong issuer/audience, unsupported algorithms, public-client secrets used as authentication, revoked grants, and refresh replay.
- Test authorization independently from login: unassigned user, viewer invoking a deployment route, cross-project requests, stale assignments, disabled app, and directory administrator without project rights.
- Test password throttling and CPU/memory use, reset replay, account enumeration, session fixation, CSRF, credential leakage, unsafe branding/URLs, and accidental sibling-subdomain cookie access.
- Test Hauth restart, database restart/unavailability, clock skew, mail failure, stale JWKS cache, signing-key rollover, session revoke, and restoration from independent database/key backups.
- Test Sloyer still reads existing project configuration during Hauth outage where safe, but fails closed for unauthenticated or stale privileged operations. No Hauth outage creates an auth bypass.
- Audit dependency updates and obtain external review of the actual integrated application. A design document and a protocol library do not by themselves establish security.

## 13. Documentation to ship

Include administrator bootstrap/recovery, app registration walkthrough, Sloyer integration guide, API audience/scope/role definitions, session and revocation semantics, email/recovery setup, key rotation, database restoration, and a minimal example app. Explain the distinction between Hauth registration, Sloyer project permissions, GitHub App installation, and GHCR pull access.

## 14. Primary references

- [Microsoft application objects and service principals](https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals): product-model reference, not a compatibility claim.
- [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html): interoperable authentication contract.
- [RFC 9700 OAuth security best current practice](https://www.rfc-editor.org/rfc/rfc9700.html): secure authorization flow and threat guidance.
- [oidc-provider](https://github.com/panva/node-oidc-provider): proposed standards engine; verify maintained release support during implementation.
- [OWASP password storage](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html): password-hashing policy.
- [PostgreSQL documentation](https://www.postgresql.org/docs/current/): roles, databases, backups, and recovery.
