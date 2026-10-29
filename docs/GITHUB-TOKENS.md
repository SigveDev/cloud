# GitHub tokens for Argo read access and image promotion

Use two separate fine-grained personal access tokens, each restricted to **SigveDev/cluster**. Actual token values are never recorded here.

| Purpose | Permission | Where the value goes |
| --- | --- | --- |
| Argo CD reads private Git manifests | Contents: Read-only; Metadata read is automatic | Argo CD repository connection |
| App workflow dispatches cluster promotion | Actions: Read and write; Metadata read is automatic | hello-world repository Actions secret `CLUSTER_REPO_TOKEN` |

Neither of these fine-grained personal tokens needs Contents write, administration access, account permissions, or access to all repositories. Private node image pulls use a third, separate classic PAT with read:packages; see [private registry setup](PRIVATE-REGISTRY.md). The cluster promotion workflow commits with its own built-in GITHUB_TOKEN, which requests Contents write in its workflow YAML.

## Argo repository-read token

1. Open GitHub Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token.
2. Name it `argocd-cluster-read`; select resource owner `SigveDev` and an expiration such as 90 days.
3. Under Repository access, choose Only select repositories and select **cluster**.
4. Add the repository permission **Contents: Read-only**. Keep all other optional permissions unset. Metadata read is automatic.
5. Generate the token and copy it directly into Argo CD's HTTPS repository connection password/token field. Do not share it in chat or Git.
6. In Argo use URL `https://github.com/SigveDev/cluster.git`, username `SigveDev`, and this token as the password. Confirm connection success.

## CLUSTER_REPO_TOKEN for cross-repository promotion

Before testing, push the updated cluster workflow to main and publish the hello-world image. Keep its package private: grant cluster Read access under Manage Actions access, and provision the Kubernetes ghcr-pull Secret as described in [private registry setup](PRIVATE-REGISTRY.md). The dispatch token is not an image-pull credential.

1. Return to GitHub's fine-grained token creation page and create a separate token named `hello-world-cluster-dispatch`.
2. Set resource owner **SigveDev**, choose an expiration, and select **Only select repositories → cluster**.
3. Under Repository permissions, add **Actions → Read and write**. Do not confuse Actions with the separate Workflows permission.
4. Leave Contents, Workflows, Administration, and all other optional permissions unset. Metadata read is automatic.
5. Generate and copy the token. It can run workflows in cluster, so restrict edits to the app workflow that uses it.
6. Open **SigveDev/hello-world → Settings → Secrets and variables → Actions**.
7. Select **New repository secret**. Use the exact name **CLUSTER_REPO_TOKEN**, without an extra prefix, and paste the token into Secret.
8. Click Add secret. Store the secret in **hello-world**, although the token's target repository is **cluster**.

The configured app workflow already references `secrets.CLUSTER_REPO_TOKEN`; no workflow edit is needed. Generate a new token and update the secret before expiration.

## Test the handoff

1. In hello-world's Actions tab, select Hello world - build and publish → Run workflow on main. A manual run starts with the current secret value.
2. Check build/smoke-test and publication. The final step should request promotion rather than report that promotion is unconfigured.
3. In cluster's Actions tab, confirm Promote hello-world starts and completes. Its own token commits the digest into `apps/hello-world/deployment.yaml`.
4. Confirm the cluster Git commit references the new image digest. Refresh Argo CD and manually Sync hello-world, unless Auto-Sync is enabled.
5. Check the deployed image and the trusted public HTTPS endpoint. Dispatch success alone is not proof that promotion or deployment succeeded.

## Common errors

- **404 dispatch:** check the token's selected cluster repository, its owner, that the cluster workflow filename matches, and that it is pushed to main.
- **403 dispatch:** review Actions write permission, expiry/revocation, and any account/organization token policy.
- **Promotion cannot pull image:** check GHCR package visibility or the separately configured registry read credentials.
- **Promotion cannot push Git commit:** review cluster workflow's built-in token permissions and main branch rules. Do not expand CLUSTER_REPO_TOKEN with Contents write; it does not perform the commit.
- **Argo cannot read:** check its separate read token, exact repository URL, expiration, and connection settings.

## Sources

- [GitHub fine-grained personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)
- [Workflow dispatch permission](https://docs.github.com/en/rest/actions/workflows#create-a-workflow-dispatch-event)
- [Creating Actions repository secrets](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets)
