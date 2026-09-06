# Step 6 — Cluster Autoscaler (node-level scaling)

## Why this exists, in plain terms

HPA (step 3's `09/10/11-hpa-*.yaml`) can add more *pods* — but only if an
existing node has room. Once both nodes are full, new pods just sit
`Pending` forever with nothing to do about it. Cluster Autoscaler is the
piece that watches for exactly that ("a pod can't be scheduled because no
node has room") and reacts by increasing your node group's desired count
(2 → up to 4, the max already set in `02-cluster/cluster.yaml`). It also
does the reverse: if nodes are sitting mostly empty for a while, it
safely drains and removes them, back down to the minimum of 2.

This is the same IRSA pattern as the AWS Load Balancer Controller (step
4) — a pod needs to call real AWS APIs (here, `autoscaling:*` to resize
the ASG), so it gets its own IAM role via a Kubernetes ServiceAccount,
scoped to just what it needs.

## What's already true (verified, not assumed)

`eksctl` already tagged the node group's underlying Auto Scaling Group
with the two tags Cluster Autoscaler needs to auto-discover it:
```
k8s.io/cluster-autoscaler/enabled = true
k8s.io/cluster-autoscaler/india-eks = owned
```
Nothing to do on the ASG side — this step is purely "deploy the
controller pod and give it permission."

## 6a. Create the IAM policy

`cluster-autoscaler-iam-policy.json` in this folder is based on the
official upstream policy
(`kubernetes/autoscaler/cluster-autoscaler/cloudprovider/aws/README.md`),
tightened one step further: the two *mutating* actions
(`SetDesiredCapacity`, `TerminateInstanceInAutoScalingGroup`) are scoped
with a `Condition` so they only work on an ASG tagged
`k8s.io/cluster-autoscaler/india-eks=owned` — i.e. only your own node
group, not any other ASG in the account. The read-only discovery actions
stay unscoped (they have to be, to find ASGs by tag in the first place).

```bash
aws iam create-policy \
  --policy-name ClusterAutoscalerIAMPolicy \
  --policy-document file://cluster-autoscaler-iam-policy.json
```
Note the returned `Arn`.

## 6b. Create the IAM role + Kubernetes ServiceAccount together

Same `eksctl create iamserviceaccount` pattern as step 4c:

```bash
eksctl create iamserviceaccount \
  --cluster india-eks \
  --region ap-south-1 \
  --namespace kube-system \
  --name cluster-autoscaler \
  --attach-policy-arn arn:aws:iam::590183900382:policy/ClusterAutoscalerIAMPolicy \
  --approve
```

## 6c. Deploy Cluster Autoscaler

```bash
kubectl apply -f cluster-autoscaler.yaml
```

## 6d. Verify

```bash
kubectl get pods -n kube-system -l app=cluster-autoscaler
kubectl logs -n kube-system deploy/cluster-autoscaler | tail -30
```

You want to see log lines mentioning it found your ASG
(`eks-india-app-ng-...`) and is polling it — no permission errors, no
crash loop.

## How you'll actually test this (once you're ready to load-test)

1. Push CPU usage up (via the HPA-driven pod scaling from step 3, or
   directly) until pods can't fit on the current 2 nodes.
2. Watch for `Pending` pods: `kubectl get pods -n india -w`
3. Watch Cluster Autoscaler react:
   `kubectl logs -n kube-system deploy/cluster-autoscaler -f`
4. Watch the ASG's desired count actually climb:
   `aws autoscaling describe-auto-scaling-groups --region ap-south-1 --auto-scaling-group-names eks-india-app-ng-96d03b3a-851c-8fa2-552a-973ab333a740 --query 'AutoScalingGroups[0].DesiredCapacity'`
5. Watch the new node join: `kubectl get nodes -w`

Full loop, in order: **HPA adds pods → pods can't fit → Cluster
Autoscaler adds a node → pods schedule onto it.** That's the complete,
automatic chain from "traffic increased" to "capacity increased" with no
manual step anywhere in the middle.
