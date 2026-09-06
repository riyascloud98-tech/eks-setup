# Step 1 — IAM roles (create these first)

Two IAM roles are needed before the cluster exists. Nothing here touches
your account except creating these two roles — safe to run now.

## Why two separate roles

- **Cluster role** — assumed by the *EKS control plane itself* (the managed
  API server AWS runs for you). It needs permission to manage the ENIs,
  load balancer wiring, etc. that make the control plane work.
- **Node role** — assumed by the *EC2 instances* that become your worker
  nodes. It needs permission to register itself with the cluster, pull
  container images from ECR, and write logs.

These are separate principals (`eks.amazonaws.com` vs `ec2.amazonaws.com`),
so they can't be the same role.

## 1a. Cluster role

```bash
aws iam create-role \
  --role-name india-eks-cluster-role \
  --assume-role-policy-document file://01-cluster-trust-policy.json

aws iam attach-role-policy \
  --role-name india-eks-cluster-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
```

## 1b. Node role

```bash
aws iam create-role \
  --role-name india-eks-node-role \
  --assume-role-policy-document file://02-node-trust-policy.json

# Lets the node join the cluster and run the CNI (pod networking)
aws iam attach-role-policy \
  --role-name india-eks-node-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

aws iam attach-role-policy \
  --role-name india-eks-node-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

# Lets the node pull your three images from ECR
aws iam attach-role-policy \
  --role-name india-eks-node-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

# Optional but recommended: lets you open a shell on a node via SSM
# Session Manager instead of needing SSH keys / a bastion host into a
# private subnet.
aws iam attach-role-policy \
  --role-name india-eks-node-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

## 1c. Note down the ARNs

```bash
aws iam get-role --role-name india-eks-cluster-role --query 'Role.Arn' --output text  -> arn:aws:iam::590183900382:role/india-eks-cluster-role
aws iam get-role --role-name india-eks-node-role --query 'Role.Arn' --output text -> arn:aws:iam::590183900382:role/india-eks-node-role
```
# ARN:
arn:aws:iam::590183900382:role/india-eks-cluster-role
arn:aws:iam::590183900382:role/india-eks-node-role

You'll paste both ARNs into `../02-cluster/cluster.yaml` in the next step.

There's a **third** IAM role needed later (for the AWS Load Balancer
Controller), but it can't be created yet — it needs the cluster's OIDC
provider URL, which only exists after the cluster is up. That's covered in
`../04-alb-controller/README.md`.
