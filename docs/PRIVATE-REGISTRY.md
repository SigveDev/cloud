# Keep the GHCR image private

Private Git repositories, private container images, and browser access to a website are separate controls. The image can stay private while the app remains publicly served over HTTPS.

Two readers need access: the cluster promotion workflow, and the K3s node pulling the workload image. CLUSTER_REPO_TOKEN only dispatches a workflow; it is not a registry credential.

## 1. Grant the cluster workflow read access to the package

In GitHub, open **SigveDev → Packages → hello-world → Package settings**.

Under **Manage Actions access**, select **Add repository**, add **SigveDev/cluster**, and set its role to **Read**. Keep the package Private. The hello-world repository's existing publishing access remains separate.

The updated `promote-hello-world.yml` requests `packages: read` and logs in to ghcr.io using the cluster workflow's built-in GITHUB_TOKEN. No additional Actions secret is required for this registry login. Repository-level package permission and the workflow token permission are both needed.

## 2. Create a node image-pull token

The VPS cannot use the short-lived GitHub Actions token for ongoing workload pulls. Create a separate **personal access token (classic)**:

1. GitHub Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token (classic).
2. Name: `sigve-cloud-ghcr-pull`.
3. Choose an expiration and record when rotation is due.
4. Select **read:packages** only; leave optional write/delete/repository/administration scopes unset.
5. Generate the token. The token owner must have read access to the hello-world package.

GHCR currently documents classic PAT authentication for outside-of-Actions package reads; the earlier fine-grained tokens serve different purposes. Do not reuse the Argo repository-read token or CLUSTER_REPO_TOKEN.

## 3. Create the Kubernetes pull Secret without committing it

The helper script prompts with hidden input and streams a Secret to the API. It never prints the token, puts it in a command argument, or writes it into the Git checkout.

From Windows PowerShell:

```powershell
scp E:\Github\cluster\scripts\create-ghcr-pull-secret.sh sigve-cloud:create-ghcr-pull-secret.sh
ssh sigve-cloud
```

On the VPS:

```bash
bash ~/create-ghcr-pull-secret.sh
sudo kubectl -n launchpad get secret ghcr-pull
```

Enter the classic token only at the hidden prompt. The Secret should have type `kubernetes.io/dockerconfigjson`. Do not share its contents or run a command that prints it into logs.

The Deployment already references `imagePullSecrets: [{name: ghcr-pull}]`. The Secret must exist in launchpad before the app rolls out. It is created out of band, is not in the Argo source path, and is not mounted into the container. The node/runtime uses it to pull images.

## 4. Publish the configuration update and start a new run

Commit/push the workflow login change, Deployment pull-secret reference, helper, and docs from the cluster checkout. Preserve/rebase concurrent image-promotion commits rather than overwriting them.

Start a **new** Promote hello-world run on main using the published digest. Re-running an old failed job can retain its original workflow definition, so it is not sufficient to test a workflow code change.

After promotion succeeds, refresh/sync hello-world in Argo CD. Verify Pod startup and the actual running digest. A successful image inspect in Actions does not prove Kubernetes has pull access.

## 5. Rotation and diagnosis

Run the helper again with a replacement classic token before expiration. Revoke the old credential after replacement is verified. A cached image may hide an expired pull credential until a new image or node needs a pull.

- Workflow 401/403: check the authenticated login step, packages read permission, and package Manage Actions access for cluster.
- ImagePullBackOff: check that ghcr-pull exists in launchpad, is referenced by the Deployment, and contains an unexpired classic token with actual package read access.
- Do not add registry scopes to CLUSTER_REPO_TOKEN; its only purpose is workflow dispatch.

## Sources

- [GHCR authentication and package reads](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [Package Actions access](https://docs.github.com/en/packages/learn-github-packages/configuring-a-packages-access-control-and-visibility)
- [Kubernetes private image pulls](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
