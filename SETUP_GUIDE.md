# EKS Setup Guide — india-eks

This is the full plan for putting `eks-india-root`, `eks-india-checkout`,
and `eks-india-fastapi` onto EKS behind one ALB. Written assuming ~10%
prior Kubernetes experience — every step says *why*, not just *what*.

Nothing in this folder is Terraform/CloudFormation. Everything is either
plain Kubernetes YAML (`kubectl apply -f ...`) or an `eksctl` YAML config
(`eksctl` is just a CLI that turns YAML into the right AWS API calls for
EKS specifically — narrower and simpler than a general IaC tool).

You run every command yourself, in order. Nothing here has been applied
to your AWS account yet.

---

## 1. The mental model

Three separate concerns, three separate layers:

```
Internet
   |
   v
[ ALB ]  <- one Application Load Balancer, in your PUBLIC subnets
   |         3 routing rules (host + path based), 1 cert, 1 IP
   |
   +--> www.indiafilings.in/           -> eks-india-root     (pods)
   +--> www.indiafilings.in/checkout*  -> eks-india-checkout (pods)
   +--> fastapi.indiafilings.in/       -> eks-india-fastapi  (pods)
   |
   v
[ EKS worker nodes ]  <- EC2 instances, in your PRIVATE-APP subnets
   Runs all the pods above, in a cluster named "india-eks"
   |
   v
[ ElastiCache Valkey ] <- in your PRIVATE-DB subnets, already created
   Only eks-india-fastapi talks to this (via the security group chain
   you already built: web SG -> app SG -> db SG)
```

The ALB is **not** part of Kubernetes — it's a real AWS Application Load
Balancer, created and managed *for* Kubernetes by a controller pod
running inside the cluster (step 4). One `Ingress` Kubernetes object with
3 rules = that one ALB with 3 listener rules.

## 2. What already exists in your account (verified via CLI, not assumed)

| Resource | Value |
|---|---|
| VPC | `vpc-01e4021e3eedba594` (172.31.0.0/16) |
| Public subnets (ALB) | `subnet-06b97ccb48b7eebc6` (1a), `subnet-0595dc8c2cb950f9d` (1b), `subnet-08afc29177531432e` (1c) — route to IGW |
| Private app subnets (EKS nodes) | `subnet-08696b445eb48a346` (1a), `subnet-08a5dc4d027dd3d9c` (1b), `subnet-0545f7c585e03f968` (1c) — route to NAT |
| Private DB subnets (ElastiCache) | `subnet-07c90b097a15f88aa` (1a), `subnet-0500232c2a691700f` (1b), `subnet-096dae9e48d0c2cae` (1c) — no NAT/IGW |
| SG: eks-web-india-sg | `sg-0af370affe06ecae1` — allows all from `0.0.0.0/0` (for the ALB) |
| SG: eks-india-app | `sg-0f8b6e007c2f5e53c` — allows all *from eks-web-india-sg only* |
| SG: eks-india-db | `sg-09ee16282153e163f` — allows all *from eks-india-app only* |
| ElastiCache | `india-elasticache-test`, endpoint `clustercfg.india-elasticache-test.ec5ns5.aps1.cache.amazonaws.com:6379` |
| ECR repos | `eks-india-root`, `eks-india-checkout`, `eks-india-fastapi` (all pushed, account `590183900382`, `ap-south-1`) |
| EKS cluster | **none yet** — this guide creates it |
| DNS for `indiafilings.in` | **Cloudflare** (external, not Route53) |
| ACM certificate | `*.indiafilings.in`, **ISSUED**, `arn:aws:acm:ap-south-1:590183900382:certificate/2303f522-9ec0-4740-9987-fb856d4f9565` — covers both `www` and `fastapi` (single-level wildcard) |

**Good news: your security groups already form the exact 3-tier chain
this needs (internet -> ALB -> nodes -> cache), with nothing more to
add.** That was already done correctly before this conversation.

## 3. Order of execution

Each numbered folder is one step. Do them in order — each one depends on
the previous.

| # | Folder | What it creates | Reversible? |
|---|---|---|---|
| 1 | `01-iam/` | 2 IAM roles (cluster, nodes) | Yes, cheaply |
| 2 | `02-cluster/` | The EKS cluster + node group (real EC2 instances) | Costly to redo — ~15-20 min, and billed from creation |
| 3 | `03-k8s/` | Namespace, config, the 3 apps' Deployments/Services | Yes, cheaply |
| 4 | `04-alb-controller/` | The controller that turns Ingress -> real ALB | Yes, cheaply |
| 5 | `05-ingress/` | The Ingress -> creates the actual ALB + DNS routing | Creates a real ALB (billed) |

## 4. Judgment calls — the things you asked me to just decide

**Managed Node Group vs. Karpenter:** going with a plain **managed node
group** (a fixed-shape pool of EC2 instances EKS manages for you). Karpenter
is a more advanced autoscaler that provisions exactly-sized nodes
on demand — genuinely better at scale, but it's another moving part with
its own IAM role, CRDs, and provisioner YAML to learn. At 3 small apps and
10% prior k8s experience, that complexity isn't paying for itself yet.
Revisit this once you have real traffic patterns and want to optimize
node cost/bin-packing.

**Helm:** used for exactly one thing — installing the AWS Load Balancer
Controller (step 4). That's genuinely the standard, AWS-documented way to
install it; everything else in this setup is plain `kubectl apply -f`.
You are not adopting Helm as your general deployment tool here.

**Node instance type:** `t3.medium` x 2 (min 2, max 4) to start. This is
a number to revisit, not a permanent decision — `kubectl top nodes` after
a week of real traffic will tell you if it's over- or under-sized.

## 5. DNS — Cloudflare, after the ALB exists

DNS for `indiafilings.in` is on Cloudflare, and the ACM certificate
(`*.indiafilings.in`) is already issued — so both files in `05-ingress/`
already have the real certificate ARN filled in, nothing pending there.

`05-ingress/` is now **two** Ingress objects, not one — split after
discovering eks-india-checkout needed its own `/checkout/healthz` health
check path (once its `basePath` fix landed), which isn't settable
per-backend within a single Ingress. `group.name: india-web` on both
merges them into one real ALB regardless; `group.order` (10 vs 20)
guarantees the more-specific `/checkout` rule is still evaluated before
the catch-all `/`. Apply both:

```bash
kubectl apply -f 05-ingress/10-ingress-checkout.yaml
kubectl apply -f 05-ingress/20-ingress-main.yaml
kubectl get ingress -n india
# ADDRESS column fills in after a minute or two, e.g.:
# india-web-1234567890.ap-south-1.elb.amazonaws.com
# (same address on both rows — one shared ALB)
```

Take that ALB DNS name and, in Cloudflare, create two CNAME records:

| Name | Target | Proxy status |
|---|---|---|
| `www` | `<the ALB DNS name>` | **DNS only** (grey cloud) |
| `fastapi` | `<the ALB DNS name>` | **DNS only** (grey cloud) |

Turn Cloudflare's proxy (orange cloud) **off** for both — a proxied
record terminates TLS at Cloudflare using Cloudflare's own certificate,
which both bypasses your ACM cert and adds a second TLS hop for no
benefit here. Since the ALB listener is already on 443 with its own
valid cert, DNS-only is correct.

## 6. Valkey credentials — answering your "is this required in Kubernetes" question

No new *concept* is required. Your team's ".env file" habit maps directly
onto two native Kubernetes objects:

- **ConfigMap** (`03-k8s/01-configmap-valkey.yaml`) — the non-secret half
  (host, port, TLS flag, CORS origins). Plain text, fine to commit.
- **Secret** (username/password) — same idea, but you create it with an
  imperative command instead of a committed file, so the plaintext
  password never touches a file on disk or git history:

  ```bash
  kubectl create secret generic valkey-credentials \
    --namespace india \
    --from-literal=VALKEY_USERNAME=riyasdeen \
    --from-literal=VALKEY_PASSWORD='<YOUR_VALKEY_PASSWORD>'
  ```

Both get injected into the FastAPI pod as environment variables via
`envFrom` in `03-k8s/03-deployment-fastapi.yaml` — the FastAPI code
already reads `os.getenv("VALKEY_HOST")` etc. (from the earlier session),
so **no code change is needed**, only *where* the values come from
changes (Kubernetes injects them instead of a local `.env` file).

Given that password was typed into this chat and now into a local `.env`
file, rotate it via the `india-eks-test` ElastiCache user group once
you're done testing.

## 7. A gotcha the ALB controller is notorious for

The AWS Load Balancer Controller auto-discovers which subnets to put the
ALB in by looking for specific tags. `eksctl` is supposed to add these
automatically to any subnet listed in `cluster.yaml`'s `vpc.subnets`
block, but if step 5's Ingress ever fails with "no subnets found" in the
controller logs, this is almost always why. Fix by tagging manually:

```bash
# Public subnets (one for the ALB)
aws ec2 create-tags --resources subnet-06b97ccb48b7eebc6 subnet-0595dc8c2cb950f9d subnet-08afc29177531432e \
  --tags Key=kubernetes.io/role/elb,Value=1 Key=kubernetes.io/cluster/india-eks,Value=shared

# Private subnets (nodes)
aws ec2 create-tags --resources subnet-08696b445eb48a346 subnet-08a5dc4d027dd3d9c subnet-0545f7c585e03f968 \
  --tags Key=kubernetes.io/role/internal-elb,Value=1 Key=kubernetes.io/cluster/india-eks,Value=shared
```
