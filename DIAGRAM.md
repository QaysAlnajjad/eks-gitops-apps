## GitOps Architecture

This repository represents the GitOps layer of the platform.

ArgoCD continuously monitors this repository and reconciles the desired state into the Kubernetes cluster.

Changes pushed to Git are automatically applied to the cluster, enabling a fully declarative and auditable deployment model.

This repository is responsible for:

  * Application deployments (Deployments, Services, Ingress)
  * Monitoring stack (Prometheus, Alertmanager, Grafana)
  * Platform-level Kubernetes resources managed via GitOps

For full infrastructure architecture (VPC, EKS, IAM, CI/CD), refer to the eks-infrastructure repository.

```text
 https://github.com/QaysAlnajjad/eks-infrastructure/blob/main/Project_Diagram.pdf
```

---

```text
                GitOps Repository
        (eks-gitops-apps - manifests)
                        │
                        │ (Git push / changes)
                        ▼
                  ArgoCD (Controller)
                        │
        ┌───────────────┼────────────────┐
        │               │                │
        ▼               ▼                ▼
   Application      Monitoring      Platform Components
   (Deployments)    (Prometheus)    (Ingress / Controllers)
        │               │
        ▼               ▼
     Pods ─────────► Prometheus
        │               │
        ▼               ▼
     Service        Alertmanager
        │               │
        ▼               ▼
     Ingress         Grafana
        │               │
        ▼               ▼
      ALB           Webhook (Telegram)
```
---
