# Production EKS Observability Stack — Prometheus & Grafana Deployment Runbook

> **CloudOps Runbook** | Category: Kubernetes / Observability | Severity: P2 | Status: Stable

A production-grade, hardened deployment runbook for standing up an Amazon EKS cluster with a fully integrated Prometheus + Grafana observability stack using the `kube-prometheus-stack` Helm chart (Prometheus Operator).

This is an **original operational document** — not a copy-paste tutorial. See the full [runbook](./runbooks/eks-prometheus-grafana-runbook.md).

---

## What This Covers

| Phase | Description |
|---|---|
| Cluster provisioning | EKS with a managed node group via `eksctl` (OIDC enabled, no SSH), dedicated VPC across multiple AZs |
| IAM & IRSA | Scoped provisioning role, IRSA for the ALB Controller and the EBS CSI driver (not `ElasticLoadBalancingFullAccess`) |
| AWS Load Balancer Controller | Helm install with an IRSA service account and the project's scoped IAM policy, pinned to a release tag |
| Observability stack | `kube-prometheus-stack` — Prometheus Operator, Grafana, Alertmanager (HA, 2 replicas) |
| Ingress & TLS | ALB Ingress with ACM certificate, HTTP→HTTPS redirect, CIDR allow-list, Route 53 alias record |
| Storage | EBS CSI add-on, encrypted default gp3 StorageClass, PVCs for Prometheus (50Gi) and Grafana (10Gi) |
| Validation gates | Step-by-step checks after every major phase |
| Rollback | Safe teardown order (Ingress → observability stack → ALB Controller and IAM → cluster) |
| Cost guidance | Monthly cost estimate for `us-west-2` |
| Escalation | Common failure patterns and the next check for each |

---

## Key Corrections vs. Common Tutorials

Most public EKS + Prometheus tutorials get these wrong. This runbook corrects all of them:

- **IRSA over node IAM roles** — ALB Controller uses a scoped IRSA service account, not a wildcard node role
- **Prometheus Operator via `kube-prometheus-stack`** — not standalone Prometheus + Grafana charts, which have no ServiceMonitor support out of the box
- **HA Alertmanager** — 2 replicas configured; single-replica Alertmanager is a silent failure point
- **`ElasticLoadBalancingFullAccess` is wrong** — the official scoped policy from `kubernetes-sigs/aws-load-balancer-controller` is used instead
- **No IMDS dependency for the controller** — region and VPC ID are passed explicitly to the Helm install
- **No plaintext secrets** — Grafana admin password generated at runtime and stored in a Kubernetes Secret
- **Grafana is not open to the internet** — the ALB only accepts an allow-listed CIDR, over HTTPS

---

## Repository Structure

```
eks-prometheus-grafana-runbook/
└── runbooks/
    └── eks-prometheus-grafana-runbook.md   # Full deployment runbook
```

---

## Prerequisites

See [Prerequisites](./runbooks/eks-prometheus-grafana-runbook.md#prerequisites) in the runbook: a scoped IAM role, the CLI tools, an ACM certificate, and DNS control for the Grafana hostname.

---

## Quick Reference — Validation After Deploy

```bash
# Cluster
kubectl get nodes -o wide

# ALB Controller
kubectl get deploy -n kube-system aws-load-balancer-controller

# Observability stack
kubectl get pods -n monitoring
kubectl get pvc -n monitoring

# Grafana ingress
kubectl get ingress -n monitoring grafana-ingress
```

All pods should be `Running`, all PVCs `Bound`, and the ingress `ADDRESS` field populated with an ALB DNS name.

---

## AWS Services Used

`EKS` · `EC2 (managed node groups)` · `ALB` · `ACM` · `IAM (IRSA)` · `EBS (gp3)` · `Route 53` · `CloudFormation`

---

## Related

- [This runbook on the portfolio site](https://rafael-gross.com/runbooks/eks-prometheus-grafana-stack)
- [fraud-detection-aws-platform](https://github.com/Rafagross/fraud-detection-aws-platform) — SSM-based private EC2 ops with no public access
- [container-runtime-runbooks-homelabs](https://github.com/Rafagross/container-runtime-runbooks-homelabs) — container runtime setup on bare metal

---

## Author

**Rafael Gross** — Cloud Operations Engineer (AWS · GCP · Kubernetes · SSM · CloudWatch)

[Portfolio](https://rafael-gross.com) · [GitHub](https://github.com/Rafagross) · [LinkedIn](https://linkedin.com/in/erafael-gross)
