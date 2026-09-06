# Step 3 — deploy the three apps

Run these from this directory (`03-k8s/`). `kubectl` should already be
pointed at `india-eks`:

```bash
aws eks update-kubeconfig --name india-eks --region ap-south-1
```

## Order matters for the first three steps

Namespace → ConfigMap → Secret, in that order, because the Deployments
reference all three by name and `namespace: india` has to exist before
anything else can be created inside it.

### 1. Namespace

```bash
kubectl apply -f 00-namespace.yaml
```

### 2. ConfigMap (non-secret Valkey config)

```bash
kubectl apply -f 01-configmap-valkey.yaml
```

### 3. Secret — imperative command, NOT `kubectl apply -f 02-secret-valkey.example.yaml`

`02-secret-valkey.example.yaml` is a **template only**, showing the shape
of what this command creates — it should never be applied directly with
real credentials in it, since a plain YAML Secret is only base64 (not
encrypted), and a file with real values in it is one `git add .` away
from landing in a repo's history permanently. Create the real Secret with
the imperative form instead, which never writes the plaintext to disk as
a file:

```bash
kubectl create secret generic valkey-credentials \
  --namespace india \
  --from-literal=VALKEY_USERNAME=riyasdeen \
  --from-literal=VALKEY_PASSWORD='<YOUR_VALKEY_PASSWORD>'
```

If you already ran this once and just need to change the password later:

```bash
kubectl delete secret valkey-credentials -n india
kubectl create secret generic valkey-credentials \
  --namespace india \
  --from-literal=VALKEY_USERNAME=riyasdeen \
  --from-literal=VALKEY_PASSWORD='<new password>'
kubectl rollout restart deployment/eks-india-fastapi -n india
```
(The rollout restart is needed because pods only read a Secret's values
once, at container start — updating the Secret alone doesn't push new
values into already-running pods.)

## 4. The three apps

Order doesn't matter between these six — they're independent of each
other:

```bash
kubectl apply -f 03-deployment-fastapi.yaml
kubectl apply -f 04-service-fastapi.yaml
kubectl apply -f 05-deployment-root.yaml
kubectl apply -f 06-service-root.yaml
kubectl apply -f 07-deployment-checkout.yaml
kubectl apply -f 08-service-checkout.yaml
```

## 5. Verify

```bash
kubectl get pods -n india
kubectl get svc -n india
```

Expect 6 pods total (2 replicas x 3 apps), all `Running`, ready count
matching container count (`1/1` for the Next.js apps, `1/1` for FastAPI).
First pull from ECR takes a few seconds; give it ~30-60s before worrying.

## If something's stuck

```bash
# ImagePullBackOff -> almost always the node IAM role's ECR permission
kubectl describe pod -n india <pod-name>

# CrashLoopBackOff -> check what the container actually logged
kubectl logs -n india deploy/eks-india-fastapi
kubectl logs -n india deploy/eks-india-root
kubectl logs -n india deploy/eks-india-checkout

# Pod stuck Pending -> usually a scheduling problem (resources, node not ready)
kubectl get events -n india --sort-by=.lastTimestamp
```

Once all 6 pods are `Running`, move on to `../04-alb-controller/README.md`.
