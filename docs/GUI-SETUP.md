# Headlamp and Argo CD on sigve.cloud

Prepared 29 October 2026 for the existing Ubuntu 26.04 / K3s VPS. Its address is represented as `<VPS_PUBLIC_IP>`; actual infrastructure addresses and credentials are not recorded.

Targets:

- `https://cluster.sigve.cloud`: Headlamp Kubernetes management UI.
- `https://argo.sigve.cloud`: Argo CD deployment UI.

This is an installation guide, not a record of changes already made to the VPS. Run Ubuntu commands as `sigve` over SSH unless a step says Windows or Hostinger. Existing K3s/Traefik must be healthy. Keep the Kubernetes API and NodePorts closed in the Hostinger firewall; only ports 80/443 need public access for these sites.

## 1. DNS in Hostinger

At the authoritative DNS provider for sigve.cloud, add:

| Type | Name | Target |
| --- | --- | --- |
| A | cluster | `<VPS_PUBLIC_IP>` |
| A | argo | `<VPS_PUBLIC_IP>` |

A working wildcard A record to this VPS can supply these answers instead. Preserve mail records. Remove or correct conflicting records only for these two hostnames; publish AAAA records only if IPv6 works end to end. Use direct DNS records during setup rather than an intervening CDN/proxy.

In Windows PowerShell:

```powershell
Resolve-DnsName cluster.sigve.cloud -Type A
Resolve-DnsName argo.sigve.cloud -Type A
```

Both must resolve to the VPS before requesting certificates. Public TCP 80 must be allowed for HTTP-01 certificate validation, and TCP 443 for the UIs.

## 2. Check the cluster and install Helm

On the VPS:

```bash
sudo kubectl get nodes
sudo kubectl -n kube-system get pods
sudo kubectl get ingressclass
sudo kubectl -n kube-system get service traefik
```

Expect a Ready node, healthy system Pods, a `traefik` IngressClass, and a Traefik LoadBalancer Service exposing 80/443. Stop and diagnose unhealthy components before installing more controllers.

Create the working directory:

```bash
mkdir -p ~/sd-cloud-platform
cd ~/sd-cloud-platform
```

Install Helm using its official installer, pinned to the release checked for this guide:

```bash
curl -fL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4 -o get-helm.sh
```

After a successful download:

```bash
sudo bash get-helm.sh --version v4.3.1
helm version --short
```

Define a helper so Helm uses the existing protected administrator kubeconfig:

```bash
h() { sudo env KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm "$@"; }
```

Run that function definition again if you reconnect or open another server shell. It does not make the kubeconfig world-readable or copy it into the repository.

Register the official project chart repositories:

```bash
h repo add jetstack https://charts.jetstack.io
h repo add headlamp https://kubernetes-sigs.github.io/headlamp/
h repo add argo https://argoproj.github.io/argo-helm
h repo update
```

## 3. Install cert-manager

Versions checked in the chart indexes for this guide: cert-manager `v1.21.3`, Headlamp `0.46.0`, Argo CD chart `10.11.6` (application `v3.5.6`). cert-manager 1.21 documents support for Kubernetes 1.36. These are pinned versions, not instructions to install whatever is newest later.

```bash
h upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --version v1.21.3 \
  --set crds.enabled=true \
  --wait --timeout 10m
```

Check readiness:

```bash
sudo kubectl -n cert-manager get pods
sudo kubectl -n cert-manager rollout status deployment/cert-manager-webhook --timeout=180s
```

## 4. Configure Let's Encrypt issuers

Replace `YOUR_EMAIL_ADDRESS` below with a real email address before applying. The email is public configuration, not a secret. Use a staging issuer first to check ACME routing without consuming production issuance limits.

```bash
cat > issuers.yaml <<'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: YOUR_EMAIL_ADDRESS
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging-account
    solvers:
      - http01:
          ingress:
            ingressClassName: traefik
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: YOUR_EMAIL_ADDRESS
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-account
    solvers:
      - http01:
          ingress:
            ingressClassName: traefik
EOF
```

Edit the placeholders:

```bash
sudo apt install nano
nano issuers.yaml
```

Save with Ctrl+O, Enter, and exit with Ctrl+X. Then:

```bash
sudo kubectl apply -f issuers.yaml
sudo kubectl wait --for=condition=Ready clusterissuer/letsencrypt-staging --timeout=180s
sudo kubectl wait --for=condition=Ready clusterissuer/letsencrypt-prod --timeout=180s
```

## 5. Install Headlamp

Headlamp requires each browser user to authenticate. Do not enable `unsafeUseServiceAccountToken`. Disable the chart's default cluster-admin binding for its backend service account; give the human login its own RBAC below.

```bash
cat > headlamp-values.yaml <<'EOF'
fullnameOverride: headlamp
clusterRoleBinding:
  create: false
config:
  unsafeUseServiceAccountToken: false
service:
  type: ClusterIP
  port: 80
resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 500m
    memory: 256Mi
EOF
```

```bash
h upgrade --install headlamp headlamp/headlamp \
  --namespace headlamp --create-namespace \
  --version 0.46.0 \
  -f headlamp-values.yaml \
  --wait --timeout 10m
```

Create a human login with read access across the cluster and management access in the `launchpad` namespace:

```bash
cat > headlamp-access.yaml <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: launchpad
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sigve-ui
  namespace: headlamp
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: sigve-ui-view
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
  - kind: ServiceAccount
    name: sigve-ui
    namespace: headlamp
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: sigve-ui-admin
  namespace: launchpad
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin
subjects:
  - kind: ServiceAccount
    name: sigve-ui
    namespace: headlamp
EOF
sudo kubectl apply -f headlamp-access.yaml
```

The `view` role deliberately excludes Secret contents and some cluster-scoped resources. Management permissions in `launchpad` include sensitive resources in that namespace. This login is not cluster-admin; use SSH/sudo for cluster-wide administration until you deliberately design broader UI access or OIDC.

## 6. Install Argo CD

Use a single-instance installation suited to this VPS. Disable Dex until adding SSO and disable notifications until needed. Traefik will terminate public HTTPS; Argo's internal HTTP service remains ClusterIP. `server.insecure` changes backend transport, not browser authentication.

```bash
cat > argocd-values.yaml <<'EOF'
global:
  domain: argo.sigve.cloud
configs:
  params:
    server.insecure: true
  cm:
    url: https://argo.sigve.cloud
    users.anonymous.enabled: "false"
  rbac:
    policy.default: role:readonly
server:
  service:
    type: ClusterIP
dex:
  enabled: false
notifications:
  enabled: false
EOF
```

```bash
h upgrade --install argocd argo/argo-cd \
  --namespace argocd --create-namespace \
  --version 10.11.6 \
  -f argocd-values.yaml \
  --wait --timeout 15m
```

```bash
sudo kubectl -n argocd get pods
sudo kubectl -n argocd get service argocd-server
```

## 7. Create HTTPS ingress routes

Both ingresses initially request staging certificates. They listen on Traefik's HTTPS entrypoint; cert-manager creates separate HTTP-01 challenge routes on port 80.

```bash
cat > gui-ingresses.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: headlamp
  namespace: headlamp
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - cluster.sigve.cloud
      secretName: headlamp-tls
  rules:
    - host: cluster.sigve.cloud
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: headlamp
                port:
                  number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - argo.sigve.cloud
      secretName: argocd-tls
  rules:
    - host: argo.sigve.cloud
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
EOF
sudo kubectl apply -f gui-ingresses.yaml
```

Wait for the staging certificates:

```bash
sudo kubectl -n headlamp wait --for=condition=Ready certificate/headlamp-tls --timeout=300s
sudo kubectl -n argocd wait --for=condition=Ready certificate/argocd-tls --timeout=300s
```

If the Certificate objects have not appeared yet, wait a few seconds and retry. If issuance times out, use the troubleshooting section before switching issuers. Staging certificates are deliberately untrusted; do not enter login credentials while browsers report an untrusted certificate.

Switch both routes to production certificates:

```bash
sed -i 's/letsencrypt-staging/letsencrypt-prod/g' gui-ingresses.yaml
sudo kubectl apply -f gui-ingresses.yaml
```

cert-manager will reissue certificates under the new issuer. Allow several minutes; an old staging Ready condition can briefly remain visible during the change. Verify the actual public endpoints without bypassing TLS checks:

```bash
curl --fail --head https://cluster.sigve.cloud
curl --fail --head https://argo.sigve.cloud
```

Do not add `-k`. If the staging certificate is still served, wait and inspect Certificate/Order/Challenge status. In the browser, confirm a trusted certificate for each hostname before logging in. Use the explicit HTTPS URLs; this configuration does not add an ordinary HTTP-to-HTTPS redirect.

## 8. Log in to both UIs

### Headlamp

On the VPS, create a short-lived login token:

```bash
sudo kubectl -n headlamp create token sigve-ui --duration=1h
```

Open `https://cluster.sigve.cloud`, choose token login, and paste the token into Headlamp. Do not put the token in Git or chat. Repeat the command when it expires. You can inspect permitted resources and manage workloads in `launchpad`; some cluster-level views/actions will be forbidden by the selected RBAC.

### Argo CD

Retrieve the initial admin password on the VPS:

```bash
sudo kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 --decode
```

Open `https://argo.sigve.cloud` and log in as `admin`. Change the password immediately in the account/user settings and save it in your password manager. After confirming login with the new password, remove the bootstrap password Secret:

```bash
sudo kubectl -n argocd delete secret argocd-initial-admin-secret
```

For a future CLI login through this standard HTTP ingress, use gRPC-Web, for example `argocd login argo.sigve.cloud --grpc-web`. Native gRPC routing would require additional controller configuration.

## 9. Deploy using the separate app and cluster repositories

The app source and Docker CI belong to **https://github.com/SigveDev/hello-world**, local `E:\Github\hello-world`. Argo, manifests, ingress, certificates, and network policies belong to **https://github.com/SigveDev/cluster**, local `E:\Github\cluster`. Both remotes were created as private empty repositories; no commit/push has been performed here.

Follow [the deployment walkthrough](DEPLOY-HELLO-WORLD.md): push cluster and app, publish the demo image, make its GHCR package pullable, promote its digest in cluster, and give Argo read-only access to the private cluster repository.

Apply `cluster/argocd/bootstrap.yaml` once on the VPS. Argo then reads branch `main` of **SigveDev/cluster**, with paths `apps/hello-world` and `networking/launchpad`, targeting `https://kubernetes.default.svc` / `launchpad`. Review/sync networking and application separately. Do not point Argo at the app-source repository.

Optional automatic handoff uses `CLUSTER_REPO_TOKEN` in the app repo, scoped to cluster with Actions write. The cluster workflow records the image digest using its own token; no kubeconfig is sent to GitHub Actions. Keep platform Helm releases under their current bootstrap owner until deliberately migrating them to GitOps.

## Troubleshooting

```bash
sudo kubectl get ingress -A
sudo kubectl get certificate,certificaterequest,order,challenge -A
sudo kubectl -n headlamp get pods,service
sudo kubectl -n argocd get pods,service
sudo kubectl -n cert-manager logs deployment/cert-manager --tail=80
```

- Certificate failure: check public A/AAAA answers, port 80, issuer email, ACME challenge status, and the `traefik` ingress class. Do not repeatedly reinstall charts.
- 404: confirm hostname, ingress class, controller health, and the HTTPS entrypoint.
- 502/503: inspect backend Service names, ready endpoints, and Pod health.
- Argo redirect loop: confirm `server.insecure` is true in argocd-cmd-params-cm and the server has restarted with it.
- Headlamp forbidden: distinguish a failed login from an RBAC-denied action. The initial account intentionally has limited management scope.
- Helm timeout: inspect Pods and events for image pulls, memory pressure, or scheduling failures before retrying.

Keep kubeconfigs, tokens, generated private keys, and real credentials out of Git and shared logs. The configuration YAML above contains no login secrets, but inspect files before committing them.

## Sources

- [Headlamp in-cluster installation](https://headlamp.dev/docs/latest/installation/in-cluster/)
- [Headlamp authentication](https://headlamp.dev/docs/latest/installation/)
- [Headlamp chart index](https://kubernetes-sigs.github.io/headlamp/index.yaml)
- [Argo CD chart index](https://argoproj.github.io/argo-helm/index.yaml)
- [Argo CD ingress configuration](https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/)
- [Argo CD initial login](https://argo-cd.readthedocs.io/en/stable/getting_started/)
- [cert-manager chart index](https://charts.jetstack.io/index.yaml)
- [cert-manager installation](https://cert-manager.io/docs/installation/helm/)
- [cert-manager compatibility](https://cert-manager.io/docs/releases/)
- [Helm installation](https://helm.sh/docs/intro/install/)
