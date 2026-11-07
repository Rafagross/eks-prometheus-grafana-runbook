# Deploy an EKS Observability Stack with Prometheus & Grafana

> **CloudOps Runbook** | Category: Kubernetes / Observability | Severity: P2 | Estimated time: 45-90 min | Status: Stable

## Summary

This runbook deploys an Amazon EKS cluster with a Prometheus + Grafana observability stack using the `kube-prometheus-stack` Helm chart (Prometheus Operator). Use it when standing up monitoring for a new EKS environment or rebuilding the observability layer from scratch.

End state: cluster, node and pod metrics scraped by Prometheus with persistent gp3 storage, Alertmanager running two replicas, and Grafana served over HTTPS behind an ALB restricted to an allow-listed CIDR.

Every phase ends with a validation gate. Do not continue to the next step until the gate passes.

---

## When to Use This Runbook

- Standing up a new EKS environment that needs Prometheus + Grafana from day one
- Rebuilding the observability stack after a cluster replacement or DR event
- Migrating from a self-managed Prometheus deployment to Prometheus Operator
- Reference when debugging a broken kube-prometheus-stack install

---

## Environment

| Field | Value |
|---|---|
| AWS Services | EKS, EC2 (managed node group), ALB, ACM, IAM (IRSA), EBS (gp3), Route 53 |
| Region | `us-west-2` (parameterized — change `REGION` once) |
| Kubernetes | A version in **standard support** (see Step 1) |
| Node OS | Amazon Linux 2023 (EKS-optimized AMI) |
| Access Method | CLI + Helm from an admin workstation or CI runner using short-lived credentials (IAM Identity Center / OIDC) |

> **Why CLI here:** EKS provisioning, Helm releases and IRSA service accounts have no practical Console-first path. The Console is used for *validation* (CloudFormation, EKS, EC2 → Load Balancers, ACM).

---

## Prerequisites

- IAM role with scoped EKS, EC2, CloudFormation and IAM (OIDC provider, roles, policies) permissions — not root, not AdministratorAccess
- aws-cli v2, kubectl (within ±1 minor of the cluster version), eksctl, helm 3.8+ (OCI support), jq, openssl
- ACM certificate issued and validated for the Grafana hostname, in the same region as the cluster
- Route 53 hosted zone (or DNS control) for the Grafana hostname

---

## Step 1: Set Environment Variables

Define everything once. Every later command references these variables.

```bash
export CLUSTER_NAME="prod-eks-01"
export REGION="us-west-2"
export K8S_VERSION="1.35"          # must be in standard support — see note below
export NODE_TYPE="m5.large"
export NODE_MIN=2
export NODE_MAX=4
export NODE_DESIRED=2
export GRAFANA_HOSTNAME="grafana.example.com"
export ACM_CERT_ARN="arn:aws:acm:us-west-2:123456789012:certificate/xxxxxxxx"
export ALLOWED_CIDR="203.0.113.0/24"   # office / VPN egress range allowed to reach Grafana
export ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
```

> [!WARNING]
> Pick a Kubernetes version that is in **standard support** on the [EKS version calendar](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html). A cluster on a version in extended support is billed at $0.60 per hour instead of $0.10 ([EKS pricing](https://aws.amazon.com/eks/pricing/)).

**Expected result:** `echo $ACCOUNT_ID` returns your 12-digit account ID.

---

## Step 2: Create the EKS Cluster

```bash
eksctl create cluster \
  --name "$CLUSTER_NAME" \
  --region "$REGION" \
  --version "$K8S_VERSION" \
  --nodegroup-name managed-workers \
  --node-type "$NODE_TYPE" \
  --node-ami-family AmazonLinux2023 \
  --nodes "$NODE_DESIRED" \
  --nodes-min "$NODE_MIN" \
  --nodes-max "$NODE_MAX" \
  --managed \
  --with-oidc \
  --ssh-access=false
```

eksctl creates a dedicated VPC across multiple AZs with tagged public and private subnets and a **single** NAT gateway (`--vpc-nat-mode Single` is the default). For an existing VPC, pass `--vpc-private-subnets` / `--vpc-public-subnets` and make sure the subnets carry `kubernetes.io/role/elb=1` (public) and `kubernetes.io/role/internal-elb=1` (private).

**Expected result:** CloudFormation stacks `eksctl-$CLUSTER_NAME-cluster` and `eksctl-$CLUSTER_NAME-nodegroup-managed-workers` reach `CREATE_COMPLETE` (Console → CloudFormation).

**Validation gate:**

```bash
kubectl get nodes -o wide
# All nodes Ready
kubectl cluster-info
```

---

## Step 3: Install the EBS CSI Driver and a Default gp3 StorageClass

Prometheus and Grafana need persistent volumes. On EKS, EBS volumes are provisioned by the **EBS CSI driver**, which is not installed by default. Without it, every PVC stays `Pending`.

```bash
# IAM role for the driver (IRSA) — role only, the add-on creates the service account
eksctl create iamserviceaccount \
  --cluster "$CLUSTER_NAME" \
  --region "$REGION" \
  --namespace kube-system \
  --name ebs-csi-controller-sa \
  --role-name "${CLUSTER_NAME}-ebs-csi" \
  --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve

# Managed add-on
eksctl create addon \
  --cluster "$CLUSTER_NAME" \
  --region "$REGION" \
  --name aws-ebs-csi-driver \
  --version latest \
  --service-account-role-arn "arn:aws:iam::${ACCOUNT_ID}:role/${CLUSTER_NAME}-ebs-csi"
```

Create `gp3-storageclass.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Delete
parameters:
  type: gp3
  fsType: ext4
  encrypted: "true"
```

```bash
kubectl apply -f gp3-storageclass.yaml
kubectl get storageclass
```

If `gp2` also shows `(default)`, remove that flag so only `gp3` is the default:

```bash
kubectl annotate storageclass gp2 storageclass.kubernetes.io/is-default-class-
```

**Validation gate:**

```bash
kubectl get pods -n kube-system -l app=ebs-csi-controller
# ebs-csi-controller pods Running
kubectl get storageclass
# gp3 (default)
```

---

## Step 4: Create the Monitoring Namespace

```bash
kubectl create namespace monitoring
kubectl label namespace monitoring environment=production
```

**Expected result:** `kubectl get ns monitoring` shows `Active`.

---

## Step 5: Install the AWS Load Balancer Controller (IRSA)

The controller gets the scoped IAM policy published by the project, not `ElasticLoadBalancingFullAccess`. Pin the policy to the controller release you install.

```bash
# Scoped IAM policy, pinned to a release tag (not main)
curl -fsSL -o alb-iam-policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v3.5.0/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://alb-iam-policy.json

# IRSA service account
eksctl create iamserviceaccount \
  --cluster "$CLUSTER_NAME" \
  --region "$REGION" \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy" \
  --approve

# Helm install (region and VPC passed explicitly so the controller does not depend on IMDS)
VPC_ID="$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$REGION" \
  --query 'cluster.resourcesVpcConfig.vpcId' --output text)"

helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName="$CLUSTER_NAME" \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region="$REGION" \
  --set vpcId="$VPC_ID"
```

> The controller project also documents **EKS Pod Identity** as an alternative to IRSA. This runbook uses IRSA, and both are supported ([installation guide](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/deploy/installation/)).

**Validation gate:**

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
# READY 2/2
kubectl get ingressclass
# alb
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=20
```

---

## Step 6: Install kube-prometheus-stack

Create the Grafana admin credentials as a Secret first, so the password never lands in Helm release values or shell history.

```bash
kubectl create secret generic grafana-admin -n monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password="$(openssl rand -base64 24)"
```

Create `kps-values.yaml`:

```yaml
grafana:
  admin:
    existingSecret: grafana-admin
    userKey: admin-user
    passwordKey: admin-password
  persistence:
    enabled: true
    storageClassName: gp3
    size: 10Gi

alertmanager:
  alertmanagerSpec:
    replicas: 2

prometheus:
  prometheusSpec:
    retention: 15d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
```

```bash
# The chart is published as an OCI artifact. Pin --version to the release you tested.
helm install kube-prometheus-stack \
  oci://ghcr.io/prometheus-community/charts/kube-prometheus-stack \
  -n monitoring \
  -f kps-values.yaml
```

Retrieve the Grafana password when needed:

```bash
kubectl get secret -n monitoring grafana-admin -o jsonpath='{.data.admin-password}' | base64 -d
```

**Validation gate:**

```bash
kubectl get pods -n monitoring
# All Running or Completed; no CrashLoopBackOff
kubectl get pvc -n monitoring
# All Bound, STORAGECLASS gp3
```

---

## Step 7: Expose Grafana via ALB Ingress with TLS

Create `grafana-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: "<ACM_CERT_ARN>"
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/inbound-cidrs: "<ALLOWED_CIDR>"
spec:
  ingressClassName: alb
  rules:
    - host: "<GRAFANA_HOSTNAME>"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kube-prometheus-stack-grafana
                port:
                  number: 80
```

```bash
sed -i "s|<ACM_CERT_ARN>|$ACM_CERT_ARN|g; s|<GRAFANA_HOSTNAME>|$GRAFANA_HOSTNAME|g; s|<ALLOWED_CIDR>|$ALLOWED_CIDR|g" grafana-ingress.yaml
kubectl apply -f grafana-ingress.yaml
```

> [!WARNING]
> Grafana is an admin surface. Keep `inbound-cidrs` limited to your office/VPN range, or switch to `scheme: internal` and reach it over VPN. Never expose it to `0.0.0.0/0`.

**Validation gate:**

```bash
kubectl get ingress -n monitoring grafana-ingress
# ADDRESS shows an ALB DNS name (2-3 minutes)
kubectl describe ingress -n monitoring grafana-ingress
# Events: no "Failed build model" errors
```

Create a Route 53 **Alias** record for `$GRAFANA_HOSTNAME` pointing to the ALB.

**Console path:** Route 53 → Hosted zones → *your zone* → Create record → Record type A → Alias → Alias to Application and Classic Load Balancer → *region* → select the ALB.

---

## Validation

| Check | How to Verify | Expected Result |
|---|---|---|
| Nodes healthy | `kubectl get nodes` | All `Ready` |
| EBS CSI driver | `kubectl get pods -n kube-system -l app=ebs-csi-controller` | `Running` |
| Default StorageClass | `kubectl get storageclass` | `gp3 (default)` |
| ALB Controller | `kubectl get deploy -n kube-system aws-load-balancer-controller` | `2/2` Ready |
| Stack pods | `kubectl get pods -n monitoring` | All `Running` / `Completed` |
| PVCs | `kubectl get pvc -n monitoring` | All `Bound` on `gp3` |
| Alertmanager HA | `kubectl get pods -n monitoring -l app.kubernetes.io/name=alertmanager` | 2 pods `Running` |
| Scrape targets | `kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090` → http://localhost:9090/targets | Targets `UP` |
| Grafana | `https://$GRAFANA_HOSTNAME` from the allowed CIDR | Login page, valid TLS certificate |

---

## Rollback

Delete the Ingress **before** uninstalling the controller. Otherwise the ALB is left orphaned and keeps billing.

```bash
# 1. Remove the Ingress and wait for the ALB to be deleted (EC2 → Load Balancers)
kubectl delete ingress -n monitoring grafana-ingress

# 2. Remove the observability stack (deleting the namespace also deletes its PVCs → EBS volumes)
helm uninstall kube-prometheus-stack -n monitoring
kubectl delete namespace monitoring

# 3. Remove the ALB Controller and its IAM
helm uninstall aws-load-balancer-controller -n kube-system
eksctl delete iamserviceaccount --cluster "$CLUSTER_NAME" --region "$REGION" \
  --namespace kube-system --name aws-load-balancer-controller
aws iam delete-policy --policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy"

# 4. Delete the cluster (destructive — confirm first; also removes the EBS CSI add-on)
eksctl delete cluster --name "$CLUSTER_NAME" --region "$REGION"
```

---

## Cost Considerations

Approximate monthly cost running 24/7 in `us-west-2` with On-Demand pricing. Validate in the [AWS Pricing Calculator](https://calculator.aws/) before you commit.

| Resource | Estimated Monthly Cost |
|---|---|
| EKS control plane (standard support, $0.10/hr) | ~$73 |
| 2× m5.large nodes | ~$140 |
| EBS gp3: 2× 80 GiB node volumes + 60 GiB Prometheus/Grafana | ~$18 |
| NAT gateway (single) + data processing | ~$35+ |
| ALB (hours + LCUs) | ~$20+ |
| **Total** | **~$285+** |

- A cluster on an extended-support version adds $0.50/hr (~$365/month).
- Use Savings Plans or Spot for non-production.
- Add VPC endpoints (ECR, STS, S3) to cut NAT data-processing charges.
- Reduce Prometheus `retention` if you don't need 15 days.

---

## Escalation

| Condition | Next Action |
|---|---|
| PVCs stuck `Pending` | Check `kubectl get storageclass` for a default and the EBS CSI controller pods; `kubectl describe pvc` for provisioning errors (usually the IRSA role) |
| ALB Controller in CrashLoopBackOff | Check the IRSA annotation on the service account and that the cluster OIDC provider exists in IAM |
| Ingress ADDRESS blank after 5 min | `kubectl describe ingress` events and controller logs; verify subnet tags and the ACM ARN/region |
| eksctl fails during CloudFormation | Console → CloudFormation → stack → Events for the first `CREATE_FAILED` |

---

## Revision History

| Date | Author | Change |
|---|---|---|
| 2026-09-28 | Rafael Gross | Published. Added EBS CSI driver + default gp3 StorageClass, Grafana admin via Secret, `ingressClassName`, HTTP→HTTPS redirect, CIDR-restricted ALB, pinned controller policy, supported Kubernetes version, control-plane cost, safe rollback order |
| 2026-05-19 | Rafael Gross | Initial version |
