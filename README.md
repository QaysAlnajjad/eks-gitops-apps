# eks-gitops-apps

GitOps repository for the Kubernetes platform and workloads running on the EKS cluster.

This repository is the declarative source of truth for platform applications, observability components, RBAC resources, and application workloads managed through ArgoCD.

## Purpose

This repository manages the Kubernetes layer after the EKS cluster has been provisioned.

It contains:

- platform applications under `apps/`
- bootstrap manifests for in-cluster access and service accounts
- monitoring resources
- RBAC manifests

This repository is continuously synced to the cluster by ArgoCD.

## Repository Structure

```text
eks-gitops-apps/
├── apps/
│   ├── aws-load-balancer-controller/
│   ├── flask-app/
│   ├── monitoring/
│   ├── sample-app/
│   └── kustomization.yaml
├── bootstrap/
│   ├── alb-controller-serviceaccount.yaml
│   └── aws-auth.yaml
├── monitoring/
│   ├── alerts/
│   ├── dashboards/
│   ├── servicemonitors/
│   ├── README.md
│   └── namespace.yaml
└── rbac/
    ├── app-role.yaml
    ├── monitoring-cluster-role.yaml
    ├── monitoring-helm-role.yaml
    └── rolebinding.yaml
