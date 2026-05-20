# Production EKS Observability Stack — Prometheus & Grafana Deployment Runbook

> **CloudOps Runbook** | Category: Kubernetes / Observability | Severity: P2 | Status: Stable

A production-grade, hardened deployment runbook for standing up an Amazon EKS cluster with a fully integrated Prometheus + Grafana observability stack using the `kube-prometheus-stack` Helm chart (Prometheus Operator).

This is an **original operational document** — not a copy-paste tutorial. It was built by identifying gaps and incorrect practices in reference material and correcting them for real production use. See [Appendix A](./runbooks/eks-prometheus-grafana-runbook.md#appendix-a--deviations-from-the-reference-source) for the full deviation log.

---

## What This Covers

| Phase | Description |
|---|---|
| Cluster provisioning | EKS with managed node groups via `eksctl`, private worker subnets, 3 AZs |
| IAM & IRSA | Scoped provisioning role, IRSA for ALB Controller (not `ElasticLoadBalancingFullAccess`) |
| AWS Load Balancer Controller | Helm install with IRSA service account, least-privilege IAM policy |
| Observability stack | `kube-prometheus-stack` — Prometheus Operator, Grafana, Alertmanager (HA, 2 replicas) |
| Ingress & TLS | ALB Ingress with ACM certificate, HTTPS-only, Route 53 ALIAS |
| Storage | EBS gp3 PVCs for Prometheus (50Gi) and Grafana (10Gi) |
| Validation gates | Step-by-step checks after every major phase |
| Rollback | Full teardown sequence (Helm → IRSA → cluster) |
| Cost guidance | Estimated monthly breakdown for `us-west-2`, cost optimization notes |
| Troubleshooting | Common failure patterns with root cause and fix |

---

## Key Corrections vs. Common Tutorials

Most public EKS + Prometheus tutorials get these wrong. This runbook corrects all of them:

- **IRSA over node IAM roles** — ALB Controller uses a scoped IRSA service account, not a wildcard node role
- **Prometheus Operator via `kube-prometheus-stack`** — not standalone Prometheus + Grafana charts, which have no ServiceMonitor support out of the box
- **HA Alertmanager** — 2 replicas configured; single-replica Alertmanager is a silent failure point
- **`ElasticLoadBalancingFullAccess` is wrong** — the official scoped policy from `kubernetes-sigs/aws-load-balancer-controller` is used instead
- **IMDSv2 enforced** — hop limit set to 1 on node launch templates
- **No plaintext secrets** — Grafana admin password generated at runtime; retrieved from Kubernetes Secret if needed
- **Private worker nodes** — nodes in private subnets only; ALB in public subnets

---

## Repository Structure

```
eks-prometheus-grafana-runbook/
└── runbooks/
    └── eks-prometheus-grafana-runbook.md   # Full deployment runbook
```

---

## Prerequisites

- `aws-cli` v2.15+, `eksctl` 0.190+, `helm` 3.14+, `kubectl` (within ±1 minor of cluster), `jq`
- IAM role with scoped EKS + EC2 + CloudFormation + IAM OIDC permissions
- ACM certificate already issued and validated for the Grafana hostname
- VPC with 2+ AZs, properly tagged public and private subnets

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

- [Portfolio site](https://rafagross.github.io) — this runbook is featured under the Kubernetes / Observability section
- [aws-cloudops-private-ec2-operations-platform](https://github.com/Rafagross/aws-cloudops-private-ec2-operations-platform) — SSM-based private EC2 ops with no public access
- [container-runtime-runbooks-homelabs](https://github.com/Rafagross/container-runtime-runbooks-homelabs) — container runtime setup on bare metal

---

## Author

**Rafael Gross** — Cloud Operations Engineer (AWS · GCP · Kubernetes · SSM · CloudWatch)

[Portfolio](https://rafagross.github.io) · [GitHub](https://github.com/Rafagross) · [LinkedIn](https://linkedin.com/in/rafaelgross)
