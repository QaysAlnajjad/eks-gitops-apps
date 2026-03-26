This repository focuses on the GitOps layer of the platform.
For full infrastructure architecture, refer to the eks-infrastructure repository: 
```text
 https://github.com/QaysAlnajjad/eks-infrastructure/blob/main/Project_Diagram.pdf
```

---


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
