# eks-setup — full replication runbook

This repo is the complete, working configuration for running three
applications (two Next.js sites, one FastAPI backend) on Amazon EKS,
behind one shared Application Load Balancer, with a full GitOps CI/CD
pipeline (GitHub Actions builds, ArgoCD deploys).

**Everything in this document actually happened, in this order, on a
real AWS account** (`590183900382`, region `ap-south-1`) — this isn't a
theoretical plan, it's a record of exactly what was built, verified at
each step. If you're replicating this in a *different* AWS account, the
specific IDs (account number, VPC ID, subnet IDs, domain name) will all
be different for you — every step below calls out exactly which values
are account-specific vs. which commands are copy-paste-safe as-is.

No Terraform, no CloudFormation. Every AWS resource here was created with
either the `aws` CLI directly, `eksctl` (a CLI purpose-built for EKS —
narrower and simpler than a general IaC tool, still just wraps normal AWS
API calls), or plain Kubernetes YAML (`kubectl apply -f ...`).

## Table of contents

0. [Architecture — the mental model](#0-architecture--the-mental-model)
1. [Prerequisites — what must exist before step 1](#1-prerequisites--what-must-exist-before-step-1)
2. [Step 1 — IAM roles for the cluster](#2-step-1--iam-roles-for-the-cluster-01-iam)
3. [Step 2 — Create the EKS cluster](#3-step-2--create-the-eks-cluster-02-cluster)
4. [Step 3 — Deploy the apps to Kubernetes](#4-step-3--deploy-the-apps-to-kubernetes-03-k8s)
5. [Step 4 — AWS Load Balancer Controller](#5-step-4--aws-load-balancer-controller-04-alb-controller)
6. [Step 5 — Ingress, the ALB, and DNS](#6-step-5--ingress-the-alb-and-dns-05-ingress)
7. [Step 6 — Cluster Autoscaler](#7-step-6--cluster-autoscaler-06-cluster-autoscaler)
8. [Step 7 — ArgoCD (GitOps)](#8-step-7--argocd-gitops-07-argocd)
9. [Step 8 — CI/CD: GitHub Actions + AWS OIDC](#9-step-8--cicd-github-actions--aws-oidc-08-cicd)
10. [Testing the whole chain, end to end](#10-testing-the-whole-chain-end-to-end)
11. [Judgment calls made along the way](#11-judgment-calls-made-along-the-way)
12. [Known gotchas](#12-known-gotchas)

---

## 0. Architecture — the mental model

```
Developer pushes code (eks-india-root / eks-india-checkout / eks-india-fastapi)
        |
        v
GitHub Actions: build image -> push to ECR (tagged with git SHA) ->
                edit one line in THIS repo (new image tag) -> commit + push
        |
        v
ArgoCD (running inside the cluster): notices this repo changed,
        applies the updated Kubernetes manifests automatically
        |
        v
Internet
   |
   v
[ ALB ]  <- one Application Load Balancer, in PUBLIC subnets
   |         host + path based routing, 1 TLS cert, 1 IP
   |
   +--> www.<domain>/           -> eks-india-root     (pods)
   +--> www.<domain>/checkout*  -> eks-india-checkout (pods)
   +--> fastapi.<domain>/       -> eks-india-fastapi  (pods)
   |
   v
[ EKS worker nodes ]  <- EC2 instances, in PRIVATE-APP subnets
   Runs all the pods above
   |
   v
[ ElastiCache Valkey ] <- in PRIVATE-DB subnets
   Only the FastAPI app talks to this
```

Five distinct systems, each doing exactly one job, connected only at
their boundaries:

| System | Job | Knows nothing about |
|---|---|---|
| GitHub Actions | Build code into a container image, push it | The Kubernetes cluster — never touches it directly |
| ArgoCD | Keep the cluster matching what's in this git repo | How images are built |
| AWS Load Balancer Controller | Turn a Kubernetes `Ingress` into a real ALB | Your application code |
| Kubernetes (EKS) | Run containers, restart failed ones, scale replicas | AWS billing, DNS, git |
| Cluster Autoscaler | Add/remove EC2 nodes when pods don't fit | Anything about a specific app |

This separation is deliberate: GitHub Actions never holds credentials
capable of changing your cluster (ArgoCD, running inside the cluster
itself, is the only thing with that power, and it only acts on what's in
git — every deploy is a git commit, fully auditable, revertible with
`git revert`).

---

## 1. Prerequisites — what must exist before step 1

This repo does **not** create your VPC, subnets, security groups, ECR
repos, ElastiCache, or ACM certificate — those existed already. To
replicate this in a new account, create these first (values shown are
from the actual account this was built on):

| Prerequisite | This account's actual value | What it's for |
|---|---|---|
| A VPC | `vpc-01e4021e3eedba594` (`172.31.0.0/16`) | Everything lives in it |
| 3 **public** subnets (one per AZ), route to an Internet Gateway | `subnet-06b97ccb48b7eebc6` (1a), `subnet-0595dc8c2cb950f9d` (1b), `subnet-08afc29177531432e` (1c) | Where the ALB's ENIs go |
| 3 **private-app** subnets (one per AZ), route to a NAT Gateway | `subnet-08696b445eb48a346` (1a), `subnet-08a5dc4d027dd3d9c` (1b), `subnet-0545f7c585e03f968` (1c) | Where EKS worker nodes (and your pods) go |
| 3 **private-db** subnets, no NAT/IGW route | (isolated, cache-only) | Where ElastiCache lives — no internet access needed or wanted |
| Security group for the ALB | `sg-0af370affe06ecae1` — inbound `0.0.0.0/0` | Public-facing |
| Security group for EKS nodes | `sg-0f8b6e007c2f5e53c` — inbound *only from the ALB's SG* | App tier, only reachable via the load balancer |
| Security group for the cache | `sg-09ee16282153e163f` — inbound *only from the node SG* | DB tier, only reachable from the app tier |
| 3 ECR repositories | `eks-india-root`, `eks-india-checkout`, `eks-india-fastapi` | Where built images are pushed |
| An ACM certificate | `*.<yourdomain>`, region **must match** where the ALB lives | TLS for the ALB's HTTPS listener |
| DNS control for your domain | This account uses **Cloudflare** (external) — could equally be Route53 | Pointing `www`/`fastapi` at the ALB once it exists |

**Local tools required on whatever machine runs these steps:**
`aws` CLI (authenticated, with permissions to create IAM roles/policies,
EKS clusters, EC2/ELB resources), `kubectl`, `eksctl`, `helm` (used for
exactly one thing — step 4), `git`.

If you don't have Chocolatey/a package manager and a tool is missing,
download the official release zip directly and drop the `.exe` anywhere
on your `PATH` — no admin rights or package manager needed. Example
(same pattern works for `helm`, `k6`, etc.):
```bash
curl -sL -o tool.zip "https://.../tool-windows-amd64.zip"
powershell -NoProfile -Command "Expand-Archive -Path tool.zip -DestinationPath tool-extract -Force"
cp tool-extract/.../tool.exe "C:/Users/<you>/bin/tool.exe"
```

---

## 2. Step 1 — IAM roles for the cluster (`01-iam/`)

Two IAM roles, needed before the cluster exists.

**Why two, not one:** the *cluster role* is assumed by the EKS control
plane itself (`eks.amazonaws.com`); the *node role* is assumed by the
EC2 instances that become worker nodes (`ec2.amazonaws.com`). Different
principals can't share a role.

```bash
cd 01-iam

# Cluster role
aws iam create-role \
  --role-name india-eks-cluster-role \
  --assume-role-policy-document file://01-cluster-trust-policy.json
aws iam attach-role-policy \
  --role-name india-eks-cluster-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

# Node role
aws iam create-role \
  --role-name india-eks-node-role \
  --assume-role-policy-document file://02-node-trust-policy.json
aws iam attach-role-policy --role-name india-eks-node-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
aws iam attach-role-policy --role-name india-eks-node-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
aws iam attach-role-policy --role-name india-eks-node-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
# Optional but recommended: shell into a node via SSM, no SSH/bastion needed
aws iam attach-role-policy --role-name india-eks-node-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Note the ARNs — you'll paste them into 02-cluster/cluster.yaml
aws iam get-role --role-name india-eks-cluster-role --query 'Role.Arn' --output text
aws iam get-role --role-name india-eks-node-role --query 'Role.Arn' --output text
```

A **third** IAM role (for the AWS Load Balancer Controller) can't be
created yet — it needs the cluster's OIDC provider, which only exists
once the cluster is up. That's step 4.

---

## 3. Step 2 — Create the EKS cluster (`02-cluster/`)

`02-cluster/cluster.yaml` is an `eksctl` ClusterConfig. Paste the two
role ARNs from step 1 into `iam.serviceRoleARN` and
`managedNodeGroups[0].iam.instanceRoleARN`, then:

```bash
eksctl create cluster -f 02-cluster/cluster.yaml
```

Takes ~15-20 minutes, and **costs money from the moment it's created**
(control plane flat fee + the EC2 nodes) — not a free, instantly
reversible step.

**What this one command does:** creates the EKS control plane in your
**existing** VPC/subnets (not a new VPC — `vpc.id` and the six subnet IDs
are given explicitly), creates one managed node group (2-4 `t3.medium`
EC2 instances in the private-app subnets), and — because
`iam.withOIDC: true` is set — creates an OIDC provider for the cluster,
which every later IRSA-based IAM role (ALB Controller, Cluster
Autoscaler) depends on.

**Verify:**
```bash
aws eks update-kubeconfig --name india-eks --region ap-south-1
kubectl get nodes                    # both should show Ready
kubectl get pods -n kube-system      # coredns, kube-proxy, aws-node all Running
```

**A gotcha to check for immediately after cluster creation:** the AWS
Load Balancer Controller (step 4) discovers subnets by tag. `eksctl`
*should* auto-tag every subnet listed in `cluster.yaml`, but if it
doesn't (verify with `aws ec2 describe-tags`), the ALB step will fail
later with "no subnets found." Fix manually if needed:
```bash
aws ec2 create-tags --resources <public-subnet-ids...> \
  --tags Key=kubernetes.io/role/elb,Value=1 Key=kubernetes.io/cluster/india-eks,Value=shared
aws ec2 create-tags --resources <private-app-subnet-ids...> \
  --tags Key=kubernetes.io/role/internal-elb,Value=1 Key=kubernetes.io/cluster/india-eks,Value=shared
```

---

## 4. Step 3 — Deploy the apps to Kubernetes (`03-k8s/`)

Order matters for the first three — everything else references
`namespace: india` by name.

```bash
cd 03-k8s

# 1. Namespace
kubectl apply -f 00-namespace.yaml

# 2. ConfigMap — non-secret config (Valkey host/port/TLS, CORS origins).
#    This is the "team already uses a .env file" habit, mapped onto a
#    native Kubernetes object: plain text, safe to commit.
kubectl apply -f 01-configmap-valkey.yaml

# 3. Secret — imperative command, NEVER `kubectl apply -f
#    02-secret-valkey.example.yaml` (that file is a template with
#    placeholder values, showing the *shape* only — a committed file
#    with real credentials in it is one `git add .` from being in your
#    repo's history forever).
kubectl create secret generic valkey-credentials \
  --namespace india \
  --from-literal=VALKEY_USERNAME=<your-valkey-username> \
  --from-literal=VALKEY_PASSWORD='<your-valkey-password>'

# 4. The three apps — order doesn't matter between these
kubectl apply -f 03-deployment-fastapi.yaml
kubectl apply -f 04-service-fastapi.yaml
kubectl apply -f 05-deployment-root.yaml
kubectl apply -f 06-service-root.yaml
kubectl apply -f 07-deployment-checkout.yaml
kubectl apply -f 08-service-checkout.yaml

# 5. Autoscaling for pods (needs Metrics Server; see step 6 for node-level scaling)
kubectl apply -f 09-hpa-root.yaml
kubectl apply -f 10-hpa-checkout.yaml
kubectl apply -f 11-hpa-fastapi.yaml
```

**Verify:**
```bash
kubectl get pods -n india        # 6 pods (2 replicas x 3 apps), all Running
kubectl get svc -n india         # 3 ClusterIP services
kubectl get hpa -n india         # 3 HPAs, reading real CPU% (not <unknown>)
```

**Why a Secret, not a committed file, for credentials:** a plain YAML
`Secret` is only base64-encoded, not encrypted — never write real values
into a file that gets committed. The Deployment references it via
`envFrom.secretRef`, so the app code (`os.getenv("VALKEY_PASSWORD")`,
etc.) needs zero changes — only *where* the value comes from changes.

**If a pod won't start:**
```bash
kubectl describe pod -n india <pod-name>   # ImagePullBackOff -> node IAM role's ECR permission
kubectl logs -n india deploy/<name>        # CrashLoopBackOff -> see what it actually logged
kubectl get events -n india --sort-by=.lastTimestamp
```

---

## 5. Step 4 — AWS Load Balancer Controller (`04-alb-controller/`)

**Why this exists:** Kubernetes has no built-in concept of an AWS ALB.
This controller is a pod that watches for `Ingress` objects (step 5) and
creates/updates the real ALB, target groups, and listener rules to
match. Without it, an `Ingress` object just sits there doing nothing.

**Why IRSA (IAM Roles for Service Accounts):** the controller needs to
call real AWS APIs (`elasticloadbalancing:*`, `ec2:Describe*`). Giving
every node's IAM role those permissions would let *any* pod on that node
do the same — too broad. IRSA scopes an IAM role to just this one
Kubernetes ServiceAccount.

```bash
cd 04-alb-controller

# Confirm the OIDC provider (created automatically by withOIDC: true in step 2)
eksctl utils associate-iam-oidc-provider --cluster india-eks --region ap-south-1 --approve

# IAM policy — alb-controller-iam-policy.json in this folder was fetched
# directly from the aws-load-balancer-controller v3.5.0 release, not
# hand-written
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://alb-controller-iam-policy.json

# IAM role + Kubernetes ServiceAccount together, in one step
eksctl create iamserviceaccount \
  --cluster india-eks --region ap-south-1 \
  --namespace kube-system --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Install the controller via its official Helm chart
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --version 3.5.0 \
  --namespace kube-system \
  --set clusterName=india-eks \
  --set region=ap-south-1 \
  --set vpcId=<YOUR_VPC_ID> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

`serviceAccount.create=false` is important — it tells the Helm chart to
use the ServiceAccount `eksctl` already created (with the IAM role
annotation on it) instead of creating a second, unprivileged one.

**Verify:**
```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
# both Running / 1/1 within a minute or two
```

---

## 6. Step 5 — Ingress, the ALB, and DNS (`05-ingress/`)

This is **two** Ingress objects, not one — split because the checkout
app needed its own health-check path (see step 3's app deploys and the
gotcha below), which isn't settable per-backend inside a single Ingress.
`alb.ingress.kubernetes.io/group.name: india-web` on both merges them
into **one shared ALB**; `group.order` (`10` vs `20`) guarantees the
more-specific `/checkout` rule is evaluated before the catch-all `/`.

```bash
kubectl apply -f 05-ingress/10-ingress-checkout.yaml
kubectl apply -f 05-ingress/20-ingress-main.yaml
kubectl get ingress -n india
# ADDRESS column fills in after a minute or two — same address on both
# rows, confirming one shared ALB
```

**DNS — a manual step, because DNS is on Cloudflare here, not Route53**
(if you're on Route53, this can be automated with an alias record and
`external-dns` instead). Take the ALB's DNS name and create two CNAME
records:

| Name | Target | Proxy status |
|---|---|---|
| `www` | `<the ALB DNS name>` | **DNS only** (grey cloud, if using Cloudflare) |
| `fastapi` | `<the ALB DNS name>` | **DNS only** (grey cloud) |

If using Cloudflare: turn its proxy **off** for both records. A proxied
record terminates TLS at Cloudflare using Cloudflare's own certificate —
that bypasses your ACM cert and adds a pointless second TLS hop, since
the ALB's own HTTPS listener (443, with the ACM cert) already handles
TLS correctly on its own.

---

## 7. Step 6 — Cluster Autoscaler (`06-cluster-autoscaler/`)

**Why this exists, separate from HPA:** the `HorizontalPodAutoscaler`s
from step 3 can add more *pods* — but only if an existing node has room.
Once all nodes are full, new pods sit `Pending` forever with nothing to
fix it. Cluster Autoscaler watches for exactly that condition and reacts
by growing the node group's desired count (up to the `maxSize` set in
step 2's `cluster.yaml`); it also safely shrinks nodes back down when
they're mostly empty.

**Prerequisite already satisfied by `eksctl`:** the node group's Auto
Scaling Group needs two tags for auto-discovery
(`k8s.io/cluster-autoscaler/enabled=true` and
`k8s.io/cluster-autoscaler/india-eks=owned`). Verify these exist —
`eksctl` adds them automatically:
```bash
aws autoscaling describe-auto-scaling-groups --region ap-south-1 \
  --auto-scaling-group-names <your-asg-name> --query 'AutoScalingGroups[0].Tags'
```

```bash
cd 06-cluster-autoscaler

# IAM policy — the mutating actions (SetDesiredCapacity,
# TerminateInstanceInAutoScalingGroup) are scoped with a Condition to
# only this cluster's own ASG, not any ASG in the account
aws iam create-policy \
  --policy-name ClusterAutoscalerIAMPolicy \
  --policy-document file://cluster-autoscaler-iam-policy.json

# Same IRSA pattern as step 4
eksctl create iamserviceaccount \
  --cluster india-eks --region ap-south-1 \
  --namespace kube-system --name cluster-autoscaler \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/ClusterAutoscalerIAMPolicy \
  --approve

kubectl apply -f cluster-autoscaler.yaml
```

**Verify:**
```bash
kubectl get pods -n kube-system -l app=cluster-autoscaler
kubectl logs -n kube-system deploy/cluster-autoscaler | grep -i "your-asg-name"
# should show it found and registered your ASG, no permission errors
```

**Full automatic chain, once this and HPA are both in place:** traffic
rises -> HPA adds pods -> if nodes are full, Cluster Autoscaler adds a
node -> new pods schedule onto it -> traffic falls -> both scale back
down. No manual step anywhere in that loop.

---

## 8. Step 7 — ArgoCD / GitOps (`07-argocd/`)

**Why ArgoCD instead of having a CI system run `kubectl apply` directly:**
that would mean handing a third-party-hosted CI runner (GitHub's own
servers) direct credentials capable of changing anything in your
cluster. With ArgoCD, only something running *inside your own cluster*
ever has that power, and it only ever does what's written in this git
repo — meaning every deployment is a git commit, fully visible in
history, and a bad deploy is undone with `git revert`, not a frantic
manual fix.

```bash
kubectl create namespace argocd

# Large CRDs in this manifest exceed kubectl's normal annotation size
# limit with plain `apply` — --server-side avoids that
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml \
  --server-side --force-conflicts

kubectl get pods -n argocd    # wait for all 7 to be Running

# Point ArgoCD at this repo's app-deployment folders
kubectl apply -f 07-argocd/app-k8s.yaml
kubectl apply -f 07-argocd/app-ingress.yaml

kubectl get application -n argocd    # both should show Synced / Healthy
```

**What each `Application` object watches, and why split this way:**
- `app-k8s.yaml` watches `03-k8s/` (the Deployments/Services/HPAs) —
  this is what changes on every code deploy
- `app-ingress.yaml` watches `05-ingress/` separately — changes far less
  often, and an Ingress mistake risks the shared ALB itself, worth being
  able to reason about and roll back independently

**One deliberate exclusion:** `app-k8s.yaml` excludes
`*.example.yaml` — `03-k8s/02-secret-valkey.example.yaml` is a template
with placeholder values, never meant to be applied. The real Secret was
created with the imperative `kubectl create secret` command in step 3,
never committed to git, so ArgoCD correctly leaves it alone (nothing in
git defines it).

**A timing note, not a bug:** ArgoCD's `automated` sync policy means it
doesn't wait for a manual "click sync" — but by default it only *checks*
git on a periodic timer (a few minutes), not instantly on push. See step
9's note on webhooks if you want that gap closed to seconds.

---

## 9. Step 8 — CI/CD: GitHub Actions + AWS OIDC (`08-cicd/`)

**The full chain, end to end:**
```
push to eks-india-root (or checkout/fastapi)
  -> GitHub Actions builds the image, tags it with the git commit SHA
     (never ":latest" — a mutable tag never changes as text, so ArgoCD
     would have nothing to detect; a unique SHA per build is what makes
     GitOps able to see "something changed")
  -> pushes it to that app's own ECR repo
  -> edits ONE line — the image tag — in this repo's matching Deployment
     YAML (03-k8s/0X-deployment-<app>.yaml)
  -> commits + pushes that one-line change to this repo (eks-setup)
  -> ArgoCD (step 8) notices and applies it
  -> Kubernetes does a normal rolling update
```

Each app repo's workflow is deliberately hardcoded to touch only its
*own* image and its *own* one Deployment file — there's no shared step
between them, so pushing to `eks-india-checkout` can never affect
`eks-india-root` or `eks-india-fastapi`, and vice versa. (ArgoCD
reinforces this independently: even though one `Application` watches all
three Deployments together, it diffs them individually — an unchanged
Deployment is a no-op apply, no restart, no disruption to the other two
apps.)

### 9a. AWS side — OIDC provider + IAM role (I did this via CLI; shown here for replication)

```bash
cd 08-cicd

# One-time per AWS account: trust GitHub's own OIDC issuer
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1

# IAM role — trusted ONLY by these specific repos on master, permitted
# to push ONLY to their own three ECR repos, nothing else
aws iam create-role \
  --role-name github-actions-ecr-push \
  --assume-role-policy-document file://github-oidc-trust-policy.json
aws iam put-role-policy \
  --role-name github-actions-ecr-push \
  --policy-name ecr-push \
  --policy-document file://github-actions-ecr-policy.json
```

**A real gotcha hit during this build, worth knowing before you hit it
too:** GitHub changed the format of the OIDC token's `sub` claim for
repositories created after **July 15, 2026** — new repos use
`repo:OWNER@OWNER-ID/REPO@REPO-ID:ref:refs/heads/BRANCH` instead of the
classic `repo:OWNER/REPO:ref:refs/heads/BRANCH`. If your repos are newer
than that cutover, `github-oidc-trust-policy.json` needs the new format
(fetch the numeric IDs with `curl https://api.github.com/repos/<owner>/<repo>`
— the `id` field, and the nested `owner.id` field). This repo's trust
policy already lists both formats, since it's harmless to allow both.
Symptom if you get this wrong: GitHub Actions fails at "Configure AWS
credentials" with `Error: Could not assume role with OIDC: Not authorized
to perform sts:AssumeRoleWithWebIdentity` — check the trust policy's
`sub` values against what your actual repos use before assuming anything
else is broken.

### 9b. GitHub side — one PAT, added as a secret in each of the 3 app repos

GitHub Actions needs permission to push a commit to *this* repo
(`eks-setup`) — that's a credential tied to a GitHub account, so it has
to be created manually, once:

1. GitHub -> Settings -> Developer settings -> **Fine-grained personal
   access tokens** -> Generate new token
2. Repository access: **Only select repositories** -> select just
   `eks-setup` (not the app repos — the workflow never needs to touch
   them, `actions/checkout` handles reading its own repo on its own)
3. Permissions -> Repository permissions -> **Contents: Read and
   write** only (do *not* grant Administration — that permission covers
   repo deletion/settings/collaborators, wildly more than "push one file
   change" needs)
4. Generate, copy the token
5. Add it as a secret named `EKS_SETUP_PAT` in **each** of the three app
   repos (Settings -> Secrets and variables -> Actions -> New repository
   secret) — same token in all three is fine, since it only grants write
   access to this one repo regardless of which app repo's workflow uses
   it

### 9c. The workflow files themselves

Each app repo has `.github/workflows/deploy.yml` — near-identical across
all three, differing only in `ECR_REPOSITORY` and
`EKS_SETUP_DEPLOYMENT_FILE`. Key points if you're writing these for a
new app:

- `permissions: id-token: write` at the workflow level — this is what
  lets the workflow request an OIDC token from GitHub at all
- `aws-actions/configure-aws-credentials@v4` with `role-to-assume` — no
  stored AWS access keys anywhere, ever
- The final step clones `eks-setup` fresh (using the PAT), `sed`-replaces
  the one `image:` line, commits, pushes

### Optional: closing the ArgoCD polling gap with a webhook

By default ArgoCD checks git on a timer (a few minutes), not the instant
a commit lands — a real (small) delay between "CI pushed" and "cluster
updated." A GitHub webhook on the `eks-setup` repo, pointed at ArgoCD's
`/api/webhook` endpoint, collapses that to seconds by pushing the
notification instead of ArgoCD having to poll for it. Not set up in this
build yet — worth adding once the ALB/ArgoCD server has a stable
reachable address.

---

## 10. Testing the whole chain, end to end

```bash
# 1. Make any small code change in one app repo, e.g. eks-india-fastapi
git add -A && git commit -m "test" && git push

# 2. Watch it build (GitHub -> that repo -> Actions tab)

# 3. Watch eks-setup get an automated commit
git -C eks-setup log --oneline -3
#    should show a new "deploy: <app>@<sha>" commit at the top

# 4. Watch ArgoCD notice and sync (may take a few minutes — see the
#    webhook note above — or force it immediately):
kubectl patch application india-k8s -n argocd --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
kubectl get application -n argocd india-k8s -w

# 5. Watch the pods roll
kubectl get pods -n india -w

# 6. Confirm live
curl https://fastapi.<yourdomain>/
```

To prove autoscaling specifically works (not just that it's installed):
generate real load (e.g. `k6 run --vus 100 --duration 5m ...` against
one URL), then watch `kubectl get hpa -n india -w` (replica count
climbing), `kubectl get pods -n india -w` (new pods), and
`kubectl logs -n kube-system deploy/cluster-autoscaler -f` (its
reasoning, if pods ever can't fit on existing nodes).

---

## 11. Judgment calls made along the way

**Managed Node Group, not Karpenter:** a plain managed node group (fixed
EC2 pool EKS manages for you) was chosen over Karpenter (a more advanced,
exactly-sized-node autoscaler) — Karpenter is genuinely better at scale,
but it's another moving part (its own IAM role, CRDs, provisioner
config) that wasn't worth the complexity at 3 small apps and early-stage
traffic. Revisit once real traffic data exists.

**Helm used for exactly one thing** — installing the AWS Load Balancer
Controller (the standard, AWS-documented way to install it). Everything
else here is plain `kubectl apply -f`; this isn't an adoption of Helm as
a general deployment tool.

**GitHub Actions + ArgoCD, not AWS CodePipeline/CodeBuild:** since the
code lives on GitHub already, CodeBuild/CodePipeline would just be a
second, redundant "on push, do a build" system sitting next to the one
already in place — nothing they'd add isn't already covered.

**Node instance type (`t3.medium` x 2, min 2 max 4) and HPA ceilings
(`maxReplicas: 6`)** are starting points, not calculated from real
traffic projections — revisit both once real usage numbers exist.

---

## 12. Known gotchas

- **ALB subnet auto-discovery tags** — `eksctl` *should* add these
  automatically but verify (see step 2's note); missing tags surface as
  "no subnets found" in the ALB controller's logs at step 5.
- **Per-backend health-check paths** — `alb.ingress.kubernetes.io/healthcheck-path`
  applies to an entire Ingress object, not per-backend. If one app in a
  shared ALB needs a different health path than the others (e.g. it runs
  behind a `basePath` prefix), it needs its own Ingress object, merged
  back into the same ALB via `group.name` (see step 5).
- **`basePath` and asset routing for a path-mounted Next.js app** — if
  an app is served at `yourdomain.com/some-path/*` behind ALB path-based
  routing, it needs `basePath: "/some-path"` in `next.config.js`.
  Without it, the app's own `/_next/static/...` asset requests have no
  distinguishing prefix, collide with whatever's mounted at `/`, and 404
  in the browser even though the page itself loads fine. This also moves
  the app's health-check endpoint to `/some-path/healthz` — update the
  Dockerfile `HEALTHCHECK`, the Deployment's readiness/liveness probes,
  and that app's dedicated Ingress object's `healthcheck-path` to match.
- **GitHub OIDC subject claim format** — see step 9a; repos created
  after July 15, 2026 use a different `sub` format than older
  tutorials/examples assume.
- **`imagePullPolicy: Always` vs. immutable tags** — needed while
  everything used `:latest` (so a pod restart would re-check for a new
  image); once CI tags every build with a unique SHA, `IfNotPresent`
  (the default) works exactly as well and avoids unnecessary re-pulls —
  not urgent to change, just no longer necessary.
- **AWS Console "EKS Capabilities" (e.g. "Create Argo CD" button) showing
  "not created"** even when ArgoCD is genuinely running — that console
  feature only tracks installations made *through* it; a manual
  `kubectl apply` install (what this repo does) is fully real and
  functional, just invisible to that particular tracking UI.
