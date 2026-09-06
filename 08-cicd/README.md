# Step 8 — CI/CD (GitHub Actions + ArgoCD GitOps)

## The full picture, in order

```
developer pushes code to eks-india-root (or checkout/fastapi)
        |
        v
GitHub Actions (in that repo):
  1. builds the Docker image
  2. pushes it to ECR, tagged with the git commit SHA
  3. edits ONE line in eks-setup's Deployment YAML (the image tag)
  4. commits + pushes that one-line change to eks-setup
        |
        v
ArgoCD (running inside the cluster, watching eks-setup):
  notices the git change within minutes (or instantly, once a
  webhook is added) and re-applies the Deployment automatically
        |
        v
Kubernetes does a normal rolling update — new pods with the new image
come up, old ones terminate, zero manual steps anywhere in this chain
```

GitHub Actions never touches the cluster. ArgoCD never builds anything.
Each does exactly one job, and the two are connected only through git —
which means the entire deploy history is just `git log` on `eks-setup`.

## What's already done (I did this directly, verified working)

- ✅ ArgoCD installed in the cluster (`argocd` namespace, all pods
  `Running`)
- ✅ Two ArgoCD `Application` objects applied (`07-argocd/app-k8s.yaml`,
  `07-argocd/app-ingress.yaml`), both showing `Synced` / `Healthy`
- ✅ GitHub's OIDC provider added to this AWS account
  (`token.actions.githubusercontent.com`)
- ✅ IAM role `github-actions-ecr-push` created — trusted *only* by
  these three specific repos on `master` (not any other repo, not any
  other branch), permitted to push images *only* to their own three ECR
  repos, nothing else
- ✅ `.github/workflows/deploy.yml` written into all three app repos
  (not yet pushed to GitHub — see below)

## The one thing only you can do: a GitHub token for the "update
eks-setup" step

GitHub Actions needs permission to push a commit to `eks-setup`. That's
a credential tied to a GitHub *account*, not something AWS or I can
create — you need to generate it:

1. GitHub → Settings → Developer settings → **Fine-grained personal
   access tokens** → Generate new token
2. Resource owner: your account. Repository access: **Only select
   repositories** → `eks-setup`
3. Permissions → Repository permissions → **Contents: Read and write**
   (this is the only permission it needs)
4. Set an expiration (90 days is reasonable — you'll regenerate and
   re-paste it when it expires)
5. Generate, copy the token (starts with `github_pat_...`)

Then add it as a secret in **each of the three app repos** (Settings →
Secrets and variables → Actions → New repository secret):
- Name: `EKS_SETUP_PAT`
- Value: the token from above

Same token, pasted into all three repos — it only grants write access to
`eks-setup`, nothing else, so reusing it across the three CI workflows
is fine.

## Pushing the workflow files themselves

I wrote `.github/workflows/deploy.yml` into all three app repos locally,
but haven't pushed them — that's a commit to your GitHub repos, your
call to make. Once you've added the `EKS_SETUP_PAT` secret to all three,
tell me and I'll push all three (or you can `git push` them yourself
from each repo).

## Testing the whole chain, end to end

1. Make any small code change in, say, `eks-india-fastapi` (e.g. tweak
   the `/` endpoint's response text)
2. `git push`
3. Watch: GitHub → that repo → Actions tab (the workflow running)
4. Watch: `eks-setup`'s commit history (a new automated commit should
   appear, bumping the image tag)
5. Watch ArgoCD notice and sync:
   ```bash
   kubectl get application -n argocd india-k8s -w
   ```
6. Watch the pods roll:
   ```bash
   kubectl get pods -n india -w
   ```
7. Confirm the change is live: `curl https://fastapi.indiafilings.in/`

## One optional cleanup, not urgent

The Deployments still have `imagePullPolicy: Always`, which was needed
back when everything used the mutable `:latest` tag (so Kubernetes would
re-check for a new image on every pod restart). Now that CI tags every
build with a unique git SHA, each tag only ever points at one specific
image forever — so `imagePullPolicy: IfNotPresent` (the default) would
work exactly as well, and avoids needlessly re-pulling an already-cached
image on routine pod restarts unrelated to a deploy. Not wrong to leave
it as `Always`, just a small unnecessary cost — happy to change it if you
want.
