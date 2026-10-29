# sigve.cloud Kubernetes learning and hosting plan

Updated 29 October 2026. This is the ongoing roadmap. The user has installed Ubuntu and reported K3s running; GUI procedures have been supplied, and the first Docker/GitOps application is implemented locally. See [README](README.md) and [setup history](docs/SETUP-HISTORY.md) for reported versus independently verified state. This workspace has not deployed to the VPS or pushed to GitHub.

## Goal and platform decision

Use **Ubuntu Server 26.04 LTS + K3s** on the Hostinger VPS to learn Kubernetes while deploying real applications under `sigve.cloud`. Use `cluster` for reproducible infrastructure, manifests, GitOps configuration, and network rules; keep the standalone `hello-world` app separate and retain `sd-cloud` as the documentation workspace.

K3s is a Kubernetes distribution. You will work with standard Kubernetes resources and tools, while its bundled components reduce the work required to get a small cluster running. The user already has Argo CD experience, so basic manual exercises are optional; the immediate milestone is Docker image publication and GitOps deployment. A later kubeadm lab can teach control-plane bootstrapping that K3s packages for you.

The earlier Docker Compose and host Nginx design is replaced by Kubernetes Deployments, Services, ingress, and cert-manager. K3s includes containerd; a separate Docker Engine installation is unnecessary on the VPS. Dockerfiles and Docker-compatible images remain useful for application builds in GitHub Actions.

Ubuntu 26.04 LTS remains the selected OS. Hostinger lists it as a plain OS template. K3s documents support for modern Linux generally; that is not a specific certification of every K3s release on Ubuntu 26.04. Before installation, select a maintained K3s release, review its release notes, and validate networking, DNS, storage, and reboot behavior on this OS. Record exact versions rather than relying on an unpinned install command.

The standalone app is `E:\Github\hello-world` → `https://github.com/SigveDev/hello-world.git`. The separate infrastructure repo is `E:\Github\cluster` → `https://github.com/SigveDev/cluster.git`; both private empty GitHub remotes have now been created and verified. The original `sd-cloud` folder keeps documentation only. All actual infrastructure addresses and credentials are omitted.

Selected management interfaces: Headlamp at `cluster.sigve.cloud` and Argo CD at `argo.sigve.cloud`. Follow [the GUI installation guide](docs/GUI-SETUP.md) for DNS, pinned chart installations, HTTPS, and login configuration.

## Selected stack

| Layer | Initial choice | What you learn |
| --- | --- | --- |
| Host OS | Ubuntu Server 26.04 LTS, plain template | Linux administration, SSH, firewalling, systemd |
| Kubernetes | Single-server K3s with bundled containerd | API objects, reconciliation, scheduling, cluster operations |
| Networking | Bundled Flannel, CoreDNS, NetworkPolicy controller | Pod networking, service discovery, traffic isolation |
| Public HTTP/HTTPS | Bundled Traefik and ServiceLB | Ingress, Services, TLS termination |
| Certificates | cert-manager and Let's Encrypt | Controllers, CRDs, issuers, certificate lifecycle |
| Application definitions | Plain YAML, then Kustomize | Declarative configuration and environment overlays |
| Platform packages | Helm with pinned chart versions | Charts, values, release lifecycle |
| Image builds | GitHub Actions and GHCR | CI, image digests, registry authentication |
| Continuous delivery | Argo CD after manual deployment works | GitOps, drift, reconciliation, release promotion |
| Initial persistent storage | K3s local-path provisioner | PVCs, StorageClasses, node-local storage limitations |
| Observability | kubectl/logs/events first, Prometheus/Grafana later | Diagnosis, metrics, alerts, capacity planning |

K3s already includes components such as CoreDNS, Traefik, local-path storage, and metrics-server. Inspect the chosen release before adding duplicate controllers.

### Nginx and routing

Use bundled Traefik as the first cluster edge proxy. Do not install host Nginx on ports 80/443 alongside it. You can run Nginx inside a Pod to serve a static site; Traefik routes the hostname to that application's Service.

If learning an NGINX-based edge is important, a later exercise can replace Traefik with **NGINX Gateway Fabric** after checking release compatibility and exposure requirements. The community `kubernetes/ingress-nginx` project retired in March 2026; it is distinct from maintained NGINX products and should not be selected for a new installation.

Learn standard Ingress first for a small working example. Then enable Gateway API support in the selected Traefik version and migrate that example to Gateway/HTTPRoute resources. Verify required CRDs and certificate integration before switching traffic. Avoid adopting controller-specific routing resources until you understand the standard APIs.

## Architecture

```text
Application code on GitHub
  -> GitHub Actions: test, build, publish
  -> GHCR: immutable image digest
  -> reviewed image update in sd-cloud
  -> Argo CD inside K3s pulls desired configuration
  -> Kubernetes reconciles the application

Visitors -> DNS: app.sigve.cloud / api.sigve.cloud -> VPS IP
  -> ServiceLB exposes Traefik on TCP 80/443
  -> Ingress (later Gateway + HTTPRoute)
  -> ClusterIP Service -> ready application Pods
  -> private database Service -> database + PVC

cert-manager -> Let's Encrypt -> TLS Secret used by ingress
Administrator -> SSH tunnel or VPN -> Kubernetes API / private admin UIs
Backups -> encrypted storage outside the VPS
```

One node runs both the control plane and applications. Multiple replicas on the same VPS do not provide host-level availability. Namespaces and dev/prod overlays in this cluster are useful learning boundaries, not independent failure domains or strong isolation for untrusted tenants.

## Capacity and inventory

K3s documents a baseline server minimum of 2 CPU cores and 2 GB RAM, excluding workloads. My practical target for this project is **4 vCPU, 8 GB RAM, and 100+ GB SSD**. More memory is useful for Argo CD, databases, and a monitoring stack. These are planning estimates; inspect actual usage before adding components. Build images on GitHub to reduce VPS load.

Observed starting server: **2 vCPUs, approximately 8 GB RAM, 96 GB root filesystem, x86_64, Ubuntu 26.04.1 LTS, no swap**. Begin on this server and add components gradually; the larger target is a capacity recommendation, not a prerequisite for the initial hello-world deployment.

- [ ] Record VPS specifications, architecture, disk space, current OS, public IP, and existing workloads.
- [ ] Back up existing data before OS changes; reinstalling the Hostinger template erases it.
- [x] Identify the repository and first application: `SigveDev/hello-world` for source and `SigveDev/cluster` for deployment state, a Docker-built static Nginx welcome app at `hello.sigve.cloud`, with no database or persistent storage.
- [ ] Check the authoritative DNS provider for `sigve.cloud` and export existing records, including mail records.
- [ ] Check the selected Kubernetes minor version against supported cert-manager, Argo CD, Helm chart, and Gateway API versions. Record the version set in the repo.

## Stage 1: secure Ubuntu and install one K3s server

- [ ] Install Hostinger's plain Ubuntu 26.04 LTS template, or plan a backed-up supported upgrade/rebuild if the server uses another release.
- [ ] Set a stable hostname such as `k3s01`; configure updates, time synchronization, and planned reboots.
- [ ] Create a named sudo administrator with SSH key login. Verify a second session and console recovery before disabling password/root SSH login.
- [ ] Install basic administration tools: ca-certificates, curl, git, jq, and backup utilities.
- [ ] Review swap, cgroups, kernel, and network requirements for the selected K3s release. Keep swap disabled initially unless deliberately configuring supported swap behavior.
- [ ] Use Hostinger's firewall to expose only public TCP 80/443 and restrict SSH to trusted sources or a VPN. Keep TCP 6443 private/restricted; do not expose NodePort ranges generally.
- [ ] Configure host firewall rules in accordance with K3s networking requirements, including Pod/Service forwarding. Do not apply a generic UFW recipe that breaks cluster networking. Review effective exposure externally.
- [ ] Install a pinned K3s server release with configuration stored in `/etc/rancher/k3s/config.yaml`. Use the initial single-node SQLite datastore; record this decision for backups.
- [ ] Keep default Flannel, CoreDNS, Traefik, ServiceLB, and local-path storage initially. Select Pod/Service CIDRs that do not overlap the administrator's LAN/VPN.
- [ ] Enable Kubernetes Secret encryption at rest using the selected release's supported configuration, and document its recovery requirements.
- [ ] Protect the administrative kubeconfig and server token. Install kubectl locally and access the API through an SSH tunnel or VPN with correct TLS names; never disable certificate verification.
- [ ] Create a scoped identity for routine work after bootstrap; treat the initial kubeconfig as a cluster-administrator credential.

For a future multi-node cluster, plan private/VPN node connectivity and backend-specific firewall rules first. Never expose Flannel VXLAN UDP 8472 to the public internet.

**Milestone:** the node is Ready, system Pods are healthy, a test Pod resolves cluster DNS, and the cluster returns healthy after reboot.

## Stage 2: learn the core objects manually

Create a disposable `launchpad` namespace and a small static website, optionally served by an Nginx image.

- [ ] Write and apply a Namespace, Deployment, and ClusterIP Service using plain YAML.
- [ ] Inspect Pods, ReplicaSets, labels/selectors, endpoints, logs, events, and resource usage.
- [ ] Reach the app using `kubectl port-forward` before configuring external routing.
- [ ] Add startup/readiness/liveness probes where appropriate and resource requests/limits.
- [ ] Add a ConfigMap and a locally provisioned Secret. Learn that base64-encoded Secret data is not encryption.
- [ ] Scale replicas; delete a Pod and observe replacement. Learn why readiness determines traffic eligibility.
- [ ] Change an image, inspect rollout status/history, and perform a manual rollback.
- [ ] Diagnose intentional ImagePullBackOff, bad Service selectors, failed readiness, and unschedulable resource requests.
- [ ] Learn non-root containers, service accounts, RBAC, and an appropriate Pod Security admission policy for application namespaces.

**Milestone:** explain the path from Deployment to Pod to Service and recover an intentionally broken deployment without reinstalling the cluster.

## Stage 3: DNS, subdomains, ingress, and HTTPS

At the authoritative DNS provider, create only the records you intend to move:

| Type | Name | Value | Purpose |
| --- | --- | --- | --- |
| A | `*` | `<VPS_PUBLIC_IPV4>` | Otherwise undefined app subdomains |
| A | `app` | `<VPS_PUBLIC_IPV4>` | Optional explicit first app |
| A | `api` | `<VPS_PUBLIC_IPV4>` | Optional explicit API hostname |
| A | `@` | `<VPS_PUBLIC_IPV4>` | Root website only if moving it here |
| CNAME | `www` | `sigve.cloud` | Optional root website alias |

Preserve mail and verification records. Existing explicit records can override wildcard DNS. Add AAAA records only after IPv6 is configured end to end; review stale AAAA records for names being moved. Wildcard DNS does not create routes or certificates.

- [ ] Inspect how ServiceLB exposes Traefik on the VPS. A Kubernetes LoadBalancer Service here does not imply Hostinger supplies a managed cloud load balancer.
- [ ] Create a standard Ingress for `app.sigve.cloud` with the selected Traefik ingress class, targeting the application's ClusterIP Service.
- [ ] Verify unknown hostnames do not reach a default application. Keep databases and control dashboards internal.
- [ ] Install a compatible pinned cert-manager chart; wait for its CRDs and webhooks to be ready.
- [ ] Start with a Let's Encrypt staging issuer and HTTP-01 solver explicitly targeting the Traefik ingress class. Verify public port 80 access and ACME challenge routing.
- [ ] After staging works, issue a production certificate, configure HTTPS, and redirect ordinary HTTP traffic without breaking validation.
- [ ] Inspect Issuer/ClusterIssuer, Certificate, CertificateRequest, Order, Challenge, and TLS Secret resources; verify renewal behavior and alerts.
- [ ] Deploy a second app at `api.sigve.cloud` to learn hostname-based routing with the same public IP.
- [ ] Later migrate a sample to Gateway API with the required CRDs and supported controller/cert-manager configuration.

Use individual certificates initially. Wildcard certificates require automated DNS-01 and a verified DNS API integration; do not assume the DNS provider has a compatible built-in solver. `*.sigve.cloud` does not cover the apex or deeper names such as `a.b.sigve.cloud`.

**Milestone:** two apps have valid HTTPS on separate subdomains, with no publicly exposed application NodePorts or databases.

## Stage 4: configuration, storage, and security

- [ ] Introduce Kustomize bases and dev/prod overlays after plain manifests work. Use Helm for packaged platform components; inspect chart output and values.
- [ ] Use a namespace per app/environment where useful, and apply resource quotas/limits suited to capacity.
- [ ] Add a PostgreSQL learning workload with an appropriate maintained chart or deliberately written StatefulSet, a private Service, and a PVC.
- [ ] Inspect StorageClass, PVC, PV, reclaim policy, node affinity, and disk consumption. Test that data survives Pod recreation.
- [ ] Understand local-path volumes remain on one node: they are neither replication nor backups. Moving to another node requires a storage migration or restore.
- [ ] Apply default-deny NetworkPolicies in application namespaces, then permit DNS, ingress-to-app traffic, required app-to-database traffic, and required egress. Test both allowed and denied paths.
- [ ] Provision real secrets out of band initially; later evaluate a secrets operator or encrypted Git workflow with explicit controller integration and recovery-key handling.
- [ ] Keep Git write access and cluster-wide controller permissions restricted. Store no plaintext passwords, kubeconfigs, or tokens in the repository.

**Milestone:** application data survives replacement, unauthorized network paths fail, and you can explain how secrets and storage are protected.

## Stage 5: separate app and infrastructure GitOps repositories

The user requested independent projects under `E:\Github`:

- **hello-world:** root Dockerfile, `site/` application source, and image build/publish workflow. Remote `SigveDev/hello-world`.
- **cluster:** `argocd/bootstrap.yaml`, `apps/hello-world` deployment/Service/Certificate/Ingress, `networking/launchpad` ingress/egress policies, manifest validation, and image promotion workflow. Remote target `SigveDev/cluster`.
- **sd-cloud:** setup documentation workspace, without duplicate deployable application code.

1. Commit/push the cluster repository, then commit/push the standalone app to its existing remote.
2. App CI builds/smoke-tests and publishes `ghcr.io/sigvedev/hello-world` for the VPS's Linux/amd64 architecture.
3. Promote the digest with the cluster repo's manual workflow first. It authenticates with its GITHUB_TOKEN, checks the fixed image, and records the desired digest in Git. Keep GHCR private: grant cluster package Read access and create the namespace ghcr-pull Secret using a separate classic read:packages credential.
4. Optionally configure `CLUSTER_REPO_TOKEN` in the app repository, scoped to cluster with Actions write, so successful app builds can dispatch promotion. The app's GITHUB_TOKEN cannot write another repository.
5. Argo reads only the cluster repository, using two scoped Applications: `launchpad-networking` and `hello-world`. Start with manual sync.
6. Review and verify namespace policy (DNS, Traefik ingress, ACME solver allowance, denied general egress), then app rollout and actual HTTPS.
7. Enable app auto-sync after the initial release; decide pruning/self-healing separately. Bot commits must be compatible with branch rules; use promotion PRs if reviews are required.
8. The cluster repo also contains platform Helm values, issuer templates, GUI ingress/RBAC, and infra/k3s/config.yaml. Keep platform Helm releases under one owner until explicitly migrating them into GitOps. Adding a file in cluster does not automatically migrate a manually installed Helm release.
9. Roll back by changing the digest in cluster Git. Neither pipeline performs automatic failed-rollout rollback.
10. Keep repository access, registry pull access, and cross-repo dispatch credentials separate and out of Git.

**Milestone:** app source produces a verified image, a cluster Git commit records its digest, and Argo deploys it under the intended network restrictions.

## Stage 6: operations and failure exercises

- [ ] Begin with kubectl logs, describe, events, top, and rollout status. Then add Prometheus/Grafana with conservative retention and resource limits after checking VPS headroom. Add centralized logs only when useful.
- [ ] Configure external uptime checks, certificate expiry alerts, disk/capacity alerts, rollout failure alerts, and backup-age monitoring.
- [ ] Back up Kubernetes datastore state, the K3s server token, configuration, encryption/recovery material, and credentials needed to rebuild.
- [ ] For the initial SQLite datastore, use a consistent documented backup procedure, such as a brief scheduled K3s stop to copy state. If switching to embedded etcd, use its documented snapshot procedure. Preserve the server token with either approach.
- [ ] Back up database-native dumps and uploaded files separately; cluster datastore backups do not contain PVC application data.
- [ ] Encrypt backups off the VPS. Initial retention: 7 daily and 4 weekly; initial recovery goals: up to 24 hours of data loss and restoration within 4 hours, validated by a drill.
- [ ] Restore into a separate test environment before depending on backups. Document how to rebuild from Ubuntu, Git, registry images, credentials, and backups.
- [ ] Perform a planned node reboot and pinned K3s upgrade after reviewing version compatibility and backing up. Observe single-node downtime.
- [ ] Practice a bad release, Pod failure, lost volume restore, and certificate failure using disposable resources. Keep destructive experiments away from important data.

**Milestone:** restore an app and its data, explain a failed deployment using evidence, and upgrade/reboot without losing persistent state.

## Later learning: multiple nodes and OpenShift

Once the first cluster is understood, add a second VPS as an agent to learn scheduling, affinity, taints/tolerations, node drains, and cross-node networking. A second worker does not make the control plane highly available. Local-path volumes do not automatically follow Pods to another node. Address private networking and storage before moving stateful workloads.

For embedded-etcd control-plane HA, plan three server nodes and a stable API endpoint, plus suitable storage/backups. Multiple VMs on one physical host still share that failure domain. Compare kubeadm in a disposable multi-VM lab if learning manual control-plane setup is a goal.

OpenShift remains a later specialization. Hostinger documents no custom VPS OS-image installation, while standard single-node OpenShift provisioning installs RHCOS. Confirm a provider-supported installation path or use a different compatible environment. Kubernetes skills learned here transfer; OpenShift adds platform-specific tools, policies, Operators, Routes, and GitOps integrations. Do not install OpenShift alongside K3s on this node.

## Suggested learning order and completion criteria

Treat these as milestones rather than fixed deadlines:

1. Linux and cluster basics: secure access, Ready node, working DNS, successful reboot.
2. Kubernetes objects: manually deploy, expose internally, debug, scale, and roll back an app.
3. Networking and certificates: two HTTPS subdomains, working renewal, private admin access.
4. Stateful apps and security: PVC persistence, tested isolation, database backup and restore.
5. CI and GitOps: image build, reviewed promotion, Argo reconciliation, Git-based rollback.
6. Operations: metrics, useful alerts, recovery drill, upgrade exercise.
7. Advanced labs: Gateway API, multi-node scheduling, storage migration, kubeadm, then OpenShift if desired.

Keep short notes for each exercise: prediction, manifest change, observed behavior, diagnostic commands, and recovery. This makes the repository a learning record as well as hosting infrastructure.

## Next verification and implementation steps

- Confirm the provider firewall attachment, node/system Pod health, GUI logins, and trusted certificates; these have not been independently inspected by this workspace.
- The user has reported hello-world working. For new apps, follow the [application onboarding walkthrough](https://github.com/SigveDev/cluster/blob/main/docs/DEPLOY-NEW-APPS.md) and the separate Vite + TypeScript template. Verify each app build, package access, image digest, Argo registration, rollout, and HTTPS independently.
- Record successful live checks in [setup history](docs/SETUP-HISTORY.md), keeping credentials and actual infrastructure addresses out of Git.
- Decide whether important live apps will share the lab cluster; use a separate environment for disruptive experiments when needed.

## Primary references

- [Hostinger Ubuntu OS templates](https://www.hostinger.com/support/1583571-what-are-the-available-operating-systems-for-vps-at-hostinger/)
- [K3s requirements and firewall/network prerequisites](https://docs.k3s.io/installation/requirements)
- [K3s networking, Traefik, ServiceLB, and Gateway API](https://docs.k3s.io/networking/networking-services)
- [K3s backup and restore](https://docs.k3s.io/datastore/backup-restore)
- [Kubernetes ingress-nginx retirement](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)
- [NGINX Kubernetes projects](https://kubernetes.nginx.org/)
- [cert-manager installation](https://cert-manager.io/docs/installation/)
- [cert-manager HTTP-01](https://cert-manager.io/docs/configuration/acme/http01/)
- [Let's Encrypt challenge types](https://letsencrypt.org/docs/challenge-types/)
- [Argo CD getting started](https://argo-cd.readthedocs.io/en/stable/getting_started/)
- [GitHub image publishing](https://docs.github.com/en/actions/tutorials/publish-packages/publish-docker-images)
- [GHCR permissions and authentication](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [Hostinger DNS pointing](https://www.hostinger.com/support/1583227-how-to-point-a-domain-to-your-vps-at-hostinger/)
- [Hostinger custom OS-image policy](https://www.hostinger.com/support/8852324-can-you-add-your-own-os-image-to-vps-at-hostinger/)
