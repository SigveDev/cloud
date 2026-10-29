# Setup history and verification record

Recorded 29 October 2026 from the project conversation and local workspace inspection. This is a factual history, not an assertion that every instruction was executed. Public VPS/client addresses, passwords, private keys, and tokens are omitted.

## Platform decisions

1. The initial goal was GitHub-based application hosting on a Hostinger VPS under subdomains of `sigve.cloud`, initially considering OpenShift and Nginx.
2. A first plan proposed Ubuntu 24.04 / Docker Compose / host Nginx for simple hosting. The user selected **Ubuntu 26.04 LTS**, and that replaced the original OS recommendation.
3. The user clarified that learning Kubernetes was a primary goal. The plan changed to **Ubuntu 26.04 + K3s**, with Traefik, cert-manager, Helm, Headlamp, and Argo CD.
4. Actual OpenShift was reconsidered when its GUI was requested. The observed VPS resources are below the documented single-node requirements, and the standard CoreOS provisioning path is incompatible with Hostinger's documented custom-image policy. The user accepted a Kubernetes management GUI plus Argo CD.
5. The selected GUI hostnames are `cluster.sigve.cloud` and `argo.sigve.cloud`. The originally suggested `openshift.sigve.cloud` was not retained as a product claim.
6. The proposed namespace `learning` was replaced by **`launchpad`** before the user created workloads there. All current instructions use `launchpad`.
7. The user initially supplied **https://github.com/SigveDev/cloud**, then explicitly requested a new standalone **SigveDev/hello-world** app repo and a separate **SigveDev/cluster** infrastructure repo. The old cloud remote was left untouched; sd-cloud now retains documentation only.
8. An initial static ConfigMap-based app approach was proposed but superseded before delivery. The user explicitly required **GitHub Actions building a Docker image**, then **Argo CD deploying that image**. The supplied application implements this pipeline.

## Server access and observed inventory

- Hostinger's plain Ubuntu installation was completed by the user.
- Windows SSH initially rejected the changed server host key after the reinstall. Instructions were given to verify the ED25519 fingerprint through Hostinger Web Console before removing the old known_hosts entry. The next screenshot showed a working server shell.
- A screenshot showed Ubuntu **26.04.1 LTS**, architecture **x86_64**, **2 CPUs**, approximately **8 GB RAM**, **zero swap**, and a **96 GB root filesystem** with most of the disk free.
- Package refresh/upgrades and creation of the `sigve` sudo administrator were guided.
- A dedicated Windows Ed25519 key named `id_ed25519_sigve_cloud`, public-key installation, and key-only/sudo checks were guided. The user proceeded after that stage, but raw successful command output was not supplied.
- A Windows SSH alias named `sigve-cloud` was configured in the walkthrough. Notepad's `config.txt` filename prevented alias recognition; renaming it to the extensionless `config` was explained.
- Root/password SSH disabling and effective sshd validation were guided, followed by testing a fresh connection.
- Hostinger firewall rules were guided: client-restricted SSH, public TCP 80/443, and inbound default deny. The user showed a client-only CIDR entry; actual firewall attachment and effective network exposure were not independently inspected.
- The user requested the reboot command after Ubuntu reported a restart was required, then asked to proceed after the restart step.
- A screenshot explicitly confirmed that **UFW was disabled**. Provider firewall protection remains a separate configuration requirement.

## Kubernetes and management interfaces

- K3s was configured in the walkthrough with `node-name: k3s01`, protected kubeconfig permissions, and Secret encryption enabled. The install instructions pinned `v1.36.5+k3s1`.
- The user reported: **“ok its up now”** after the K3s installation steps. Node/Pod output and the exact installed version were not independently captured here.
- A basic manual Nginx exercise was offered. The user clarified prior Argo CD experience; the project moved directly toward management interfaces and GitOps. Do not assume the earlier exercise was deployed.
- Helm, cert-manager, staging/production ACME issuers, Headlamp, and Argo CD installation steps were supplied. Their pinned versions and values are preserved in the GUI guide.
- Headlamp login design: `headlamp/sigve-ui`, built-in cluster-wide view permissions, management permissions in `launchpad`, short-lived token login, no backend cluster-admin binding.
- Argo login design: bootstrap admin password obtained on the VPS, changed immediately, initial password Secret deleted after verifying the replacement. No actual password is documented.
- Traefik GUI ingress definitions, staging certificate checks, production switch, and browser login procedures were supplied.
- The user repeatedly continued the guide, but no final GUI screenshots or certificate/login results were provided. The live state of these services remains unverified by this workspace.

## Files created and project state

- `SETUP-PLAN.md` was created and revised as platform choices changed.
- `docs/GUI-SETUP.md` preserves the complete GUI procedure and now uses the actual repository and namespace decisions.
- The long root README, bootstrap guide, this history, and first-app deployment walkthrough document the current architecture and operational procedure.
- The hello-world app includes HTML, a health endpoint, a Dockerfile with an immutable upstream base, Kubernetes resources, and an Argo AppProject/Application definition.
- The first mixed-repository pipeline was superseded by the requested two-repo design: hello-world CI builds/smoke-tests and publishes `ghcr.io/sigvedev/hello-world`; cluster owns promotion and deployment state.
- The local folder was initially not a Git checkout. The supplied GitHub remote returned no advertised refs during inspection. The initial mixed checkout was replaced with independent hello-world and cluster local repositories on main; the documentation workspace no longer targets the old cloud origin.
- No Git commit, GitHub push, workflow run, package visibility change, DNS update, or VPS deployment is performed by the workspace implementation. Two private empty GitHub repositories, SigveDev/hello-world and SigveDev/cluster, were created and verified using authenticated GitHub API access after the user connected GitHub. Those are next actions for the user following the deployment guide.

## Verification boundaries

Local checks can render the Kustomize manifests, parse YAML and workflow scripts, and inspect the repository's configuration. They cannot establish the live VPS configuration without a connected server session, or establish successful CI publication before the user pushes.

Final local validation completed: 21 YAML/workflow files parsed; Kustomize rendered four application and four networking resources into launchpad; selectors, TLS references, Argo source paths and resource scope, network-policy selectors/allowances, workflow Python, pinned Actions, and repository permissions matched. Documentation links resolved, and source scans found no literal IPv4 addresses or recognizable private-key/GitHub-token material. The upstream image was checked for Linux/amd64 and UID 101. Docker is unavailable locally, so actual image build/runtime checks remain for GitHub Actions. No live server networking or certificate renewal test has been performed here.

Runtime smoke tests are included in GitHub Actions. Once the first release is deployed, add the date and results of these checks without credentials or actual infrastructure addresses:

- Actions build/smoke-test/publication success and source commit.
- Bot digest commit and the digest actually running in Kubernetes.
- Argo sync/health results.
- DNS checks performed privately against the real VPS address.
- Trusted certificates and browser access for all three configured hostnames.
- Backup/restore and reboot recovery results when tested.

## Final repository separation

- `E:\Github\hello-world` contains the root Dockerfile, static source, and image build workflow; origin is `SigveDev/hello-world`.
- `E:\Github\cluster` contains the Argo bootstrap, desired app manifests, namespace ingress/egress policies, validation/promotion workflows, and full setup documentation; origin is `SigveDev/cluster`.
- `E:\Github\sd-cloud` is retained as the original documentation workspace. Duplicate deployable monorepo files are removed after verifying their standalone copies.
- Browser automation could not confidently verify Chrome's URL and was stopped without submitting forms. The new remotes were subsequently created through authenticated GitHub API access, not through the stopped browser session.
- App publication and cluster promotion use separate GitHub tokens/permissions. Automatic cross-repo dispatch is optional and requires the user-provisioned CLUSTER_REPO_TOKEN; no actual credential was created or stored in Git.
- Network policies deny launchpad Pod ingress/egress by default and allow DNS, Traefik app ingress, and certificate solver validation. Live enforcement and certificate renewal remain unverified until the user applies and tests them.

## Private package follow-up

The user supplied a promotion failure showing an anonymous GHCR 401 and asked to avoid making packages public. The cluster workflow now requests packages read and logs in using its GITHUB_TOKEN. The Deployment references an out-of-band ghcr-pull Secret, and an interactive helper provisions it on the VPS using a separate classic read:packages token. Package Actions-access configuration and credential provisioning are user steps, not actions already performed here. No package visibility was changed, and no real token was read or stored by this implementation.
