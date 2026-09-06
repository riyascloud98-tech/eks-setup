# Step 4 — AWS Load Balancer Controller

## Why this exists

Kubernetes doesn't know how to create an AWS Application Load Balancer on
its own. The **AWS Load Balancer Controller** is a pod that runs inside
your cluster, watches for `Ingress` objects (step 5), and — on your
behalf — creates/updates/deletes the actual ALB, target groups, and
listener rules in your AWS account to match. Without it, the `Ingress`
YAML in step 5 just sits there doing nothing.

This step only makes sense **after** the cluster exists (step 2) — it
needs the cluster's OIDC provider, which is created as part of cluster
creation (`iam.withOIDC: true` in `cluster.yaml`).

## Why IRSA (IAM Roles for Service Accounts)

The controller pod needs to call real AWS APIs (`elasticloadbalancing:*`,
`ec2:*` for describing/creating ALB resources). The old approach was
giving *every* node's IAM role those permissions — overly broad, since
then *any* pod on that node could do the same. IRSA lets you attach an
IAM role to just this one Kubernetes ServiceAccount, so only this
controller's pod can assume it. This is why the cluster needed
`withOIDC: true`.

## 4a. Confirm the OIDC provider exists

(Created automatically by `eksctl create cluster` since `withOIDC: true`
was set — this just verifies it.)

```bash
eksctl utils associate-iam-oidc-provider --cluster india-eks --region ap-south-1 --approve
```

## 4b. Create the IAM policy the controller needs

This repo already has the official policy file, fetched directly from the
`aws-load-balancer-controller` v3.5.0 release
(`04-alb-controller/alb-controller-iam-policy.json`) — this is the exact
file AWS's own install docs point to, not something written by hand.

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://alb-controller-iam-policy.json
```

Note the returned `Arn` — you'll need it in the next command.

## 4c. Create the IAM role + Kubernetes ServiceAccount together

`eksctl create iamserviceaccount` does both in one step: creates an IAM
role trusting *only* this specific ServiceAccount (via the OIDC
provider), attaches the policy from 4b, and creates the matching
ServiceAccount object in the cluster with the right annotation already on
it.

```bash
eksctl create iamserviceaccount \
  --cluster india-eks \
  --region ap-south-1 \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::590183900382:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

## 4d. Install Helm (one-time, local tool only)

Helm is only used for this one thing in this whole setup — installing
this controller. It's not required for anything else here.

If you already have Chocolatey installed: `choco install kubernetes-helm`.
Otherwise (no package manager needed) — download the official release
zip, extract `helm.exe`, and put it anywhere on your `PATH` (e.g. next to
`eksctl.exe`):

```bash
curl -sL -o helm.zip "https://get.helm.sh/helm-v4.2.4-windows-amd64.zip"
powershell -NoProfile -Command "Expand-Archive -Path helm.zip -DestinationPath helm-extract -Force"
cp helm-extract/windows-amd64/helm.exe "C:/Users/Acer/bin/helm.exe"  # or wherever eksctl.exe already lives
helm version
```
Check https://github.com/helm/helm/releases for the current version if
`v4.2.4` is no longer latest by the time you run this.

## 4e. Install the controller

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --version 3.5.0 \
  --namespace kube-system \
  --set clusterName=india-eks \
  --set region=ap-south-1 \
  --set vpcId=vpc-01e4021e3eedba594 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

`serviceAccount.create=false` is important — it tells the chart to use
the ServiceAccount eksctl already created in 4c (with the IAM role
annotation on it), instead of creating a second, unprivileged one.

## 4f. Verify

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

Both pods should show `Running` / `1/1` within a minute or two. If they
crash-loop, `kubectl logs -n kube-system deploy/aws-load-balancer-controller`
almost always says exactly what's wrong (usually a missing IAM permission
or a subnet tagging issue — see the note in `../SETUP_GUIDE.md` about
subnet tags).
