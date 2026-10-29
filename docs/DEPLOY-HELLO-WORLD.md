# Deploy the standalone hello-world app using the cluster repository

The application source and Docker workflow live in **E:\Github\hello-world**, remote **https://github.com/SigveDev/hello-world.git**. Argo definitions, desired image versions, ingress, certificates, and namespace traffic policy live in **E:\Github\cluster**, remote **https://github.com/SigveDev/cluster.git**.

Both private, empty GitHub repositories were created and verified during setup. Both local projects are initialized on main with their own origin. Neither has been committed or pushed by the setup assistant. The pre-existing SigveDev/cloud remote was left untouched.

Real VPS/client addresses and all credentials are omitted. The original sd-cloud directory contains documentation, not the app or infrastructure code.

## 1. Commit and push cluster

In Windows PowerShell:

```powershell
Set-Location E:\Github\cluster
git status
git add .
git commit -m "Add cluster GitOps configuration and launchpad network policies"
git push -u origin main
```

Inspect the files before committing. The checked-in deployment image is a bootstrap placeholder; do not sync it yet. The cluster workflow must exist on main before automatic app-to-cluster dispatch can work.

## 2. Commit and push the app

```powershell
Set-Location E:\Github\hello-world
git status
git add .
git commit -m "Add standalone hello-world app and Docker image workflow"
git push -u origin main
```

Use your normal GitHub Git authentication. Never embed credentials in origin URLs. If the remotes gain other commits first, integrate them rather than force-pushing.

Under **SigveDev/hello-world → Actions**, wait for **Hello world - build and publish**. It builds/smoke-tests the root Dockerfile and publishes a Linux/amd64 image. Pull requests do not publish.

The run summary reports:

```text
ghcr.io/sigvedev/hello-world@sha256:<PUBLISHED_DIGEST>
```

The actual digest is release metadata, not a secret. No Kubernetes manifest is written in the app repo.

## 3. Make the demo package pullable

GHCR initially creates private packages. For this public welcome image, change the **hello-world GHCR package** visibility to Public after its first publication. This is independent of the private Git repository's visibility.

The supplied promotion workflow verifies the image anonymously, and the Kubernetes Deployment assumes a public image. If retaining a private package, provision explicit read access for the cluster workflow and an imagePullSecret in launchpad, then reference it in the Deployment. Do not commit that Secret's credential.

## 4. Promote the first image in cluster

Open **SigveDev/cluster → Actions → Promote hello-world → Run workflow**:

- Branch: main.
- image_digest: the full `sha256:...` value from the app workflow.
- source_sha: optional app commit SHA.

The workflow validates the digest, checks the fixed public image exists, updates only `apps/hello-world/deployment.yaml`, and creates a normal bot commit on cluster main. It does not run kubectl apply against your server.

Confirm the Deployment now uses an immutable reference. If branch rules block bot commits, use a PR-based promotion flow; do not force-push or broadly weaken protection. If a concurrent Git update rejects the push, rerun promotion on current main.

Bring the promotion commit into your Windows checkout before editing cluster again:

```powershell
Set-Location E:\Github\cluster
git pull --rebase origin main
```

## 5. DNS and server prerequisites

Configure an A record `hello` targeting `<VPS_PUBLIC_IP>` at sigve.cloud's authoritative DNS provider, or a suitable wildcard. Review conflicting A/AAAA records. Keep public TCP 80 for HTTP-01 and 443 for HTTPS.

In Windows PowerShell:

```powershell
Resolve-DnsName hello.sigve.cloud -Type A
```

On the VPS:

```bash
sudo kubectl get nodes
sudo kubectl get namespace launchpad
sudo kubectl get clusterissuer letsencrypt-prod
sudo kubectl -n kube-system get pods --show-labels
sudo kubectl -n cert-manager get pods --show-labels
```

The node and issuer must be Ready. If launchpad has not yet been created:

```bash
sudo kubectl create namespace launchpad --dry-run=client -o yaml | sudo kubectl apply -f -
```

Check policy selectors against actual labels: Traefik uses `app.kubernetes.io/name: traefik`, CoreDNS uses `k8s-app: kube-dns`, and the cert-manager controller uses `app.kubernetes.io/name: cert-manager`. The default K3s NetworkPolicy controller must be enabled. Do not assume a manifest render proves effective traffic filtering.

## 6. Give Argo read access to cluster

The new cluster repository is private. Under **Argo CD → Settings → Repositories**, connect:

```text
https://github.com/SigveDev/cluster.git
```

Use a dedicated read-only repository credential or read-only deploy key. Verify host keys for SSH. Do not reuse an administrator kubeconfig or the image-promotion token for repository reads. Keep actual credentials in Argo/secret storage, never in Git or chat.

Argo does not need to read the app source repo to deploy this image.

## 7. Bootstrap Argo definitions

From Windows PowerShell:

```powershell
scp E:\Github\cluster\argocd\bootstrap.yaml sigve-cloud:cluster-bootstrap.yaml
ssh sigve-cloud
```

On the VPS:

```bash
sudo kubectl apply -f ~/cluster-bootstrap.yaml
```

This creates the scoped launchpad AppProject and two Applications:

| Application | Repository | Path | Destination |
| --- | --- | --- | --- |
| hello-world | SigveDev/cluster | apps/hello-world | launchpad |
| launchpad-networking | SigveDev/cluster | networking/launchpad | launchpad |

Both use main and `https://kubernetes.default.svc`. The project allows only this source repository, this namespace, and the required resource kinds. It cannot create cluster-scoped resources. Bootstrap remains outside both managed paths.

## 8. Review and sync networking, then the application

Open https://argo.sigve.cloud. Review and manually sync **launchpad-networking** first. It applies only inside launchpad:

- default-deny ingress/egress for namespace Pods;
- DNS egress to CoreDNS;
- Traefik ingress to the hello-world Pod on 8080;
- Traefik and cert-manager ingress to temporary labeled HTTP-01 solver Pods on 8089.

The static app needs no general outbound internet access. Future apps need explicit additional allowances; these policies will also affect future Pods in launchpad. They do not replace the Hostinger firewall or configure host-wide egress.

Then inspect **hello-world**, confirm its desired image digest is the promoted value, and sync it. It creates a Deployment, ClusterIP Service, production Certificate, and HTTPS Ingress.

## 9. Verify deployment, TLS, and policy

On the VPS:

```bash
sudo kubectl -n launchpad rollout status deployment/hello-world --timeout=180s
sudo kubectl -n launchpad get pods,services,ingresses,certificates,networkpolicies
sudo kubectl -n launchpad wait --for=condition=Ready certificate/hello-world-tls --timeout=300s
sudo kubectl -n launchpad get deployment hello-world -o jsonpath='{.spec.template.spec.containers[0].image}'
curl --fail --head https://hello.sigve.cloud
```

Open https://hello.sigve.cloud. Expect the welcome page and a trusted hostname certificate. Do not bypass TLS verification. Verify allowed DNS and blocked outbound access from application Pods using appropriate diagnostic tooling. Test certificate validation/renewal with the policy in place; a ready app alone does not prove the solver exception works.

For failures:

```bash
sudo kubectl -n launchpad logs deployment/hello-world --tail=80
sudo kubectl -n launchpad get events --sort-by=.metadata.creationTimestamp
sudo kubectl -n launchpad describe certificate hello-world-tls
sudo kubectl -n launchpad get certificaterequest,order,challenge
```

ImagePullBackOff usually warrants checking package visibility/pull credentials first. DNS, issuer readiness, HTTP-01 routing, and solver policy are separate from image health.

## 10. Enable automatic handoff, if wanted

After the first manual promotion works, create a fine-grained token limited to **SigveDev/cluster**, with **Actions: write**, an expiry, and no unnecessary repository access. Store it in the **hello-world repo's Actions secrets** under `CLUSTER_REPO_TOKEN`.

This allows the app workflow to dispatch cluster's promotion workflow. It does not directly grant Contents write to the app workflow. The cluster workflow commits using its own GITHUB_TOKEN; repository/org policy and branch rules must permit it. Restrict app workflow edits because the dispatch credential can execute cluster workflows.

Without this secret, builds publish normally and report that promotion is manual. With it, check both repositories' run results: a successful request only means dispatch was accepted. The app skips dispatch if its source has already been superseded on main.

For fully automatic delivery, also enable Auto-Sync for hello-world after the first verified release. Networking can stay manually reviewed. Pruning and self-healing are separate decisions.

## 11. Change and roll back

Edit **hello-world/site/index.html** in the standalone app repo, commit, and push. Build/publication occurs there; desired image promotion occurs only in cluster.

Edit resource limits, ingress, certificates, and policies only in cluster. Pull the bot's latest commit before making infrastructure changes.

For rollback, restore a prior known-good digest in cluster's `apps/hello-world/deployment.yaml`, commit, and sync. No app rebuild is necessary. Neither pipeline automatically reverts a failed Kubernetes rollout.
