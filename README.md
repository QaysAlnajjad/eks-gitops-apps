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
```

---

## GitOps model

This repository is intended to be watched by ArgoCD.

Workflow:

1. Change manifests in Git

2. Push to repository

3. ArgoCD detects drift

4. ArgoCD syncs the cluster to the desired state

This keeps deployments declarative, traceable, and auditable.

---

## What is managed here

1. Platform applications in apps/

The apps/ directory contains ArgoCD-managed application definitions and workload manifests for:

  * AWS Load Balancer Controller

  * Flask application

  * Monitoring stack

  * Sample application

The apps/kustomization.yaml acts as the aggregation point for the application layer.

2. Monitoring resources in monitoring/

This directory holds monitoring-related Kubernetes resources separate from the core application definitions.

It includes:

  * alerts/ for alerting resources

  * dashboards/ for Grafana dashboard definitions

  * servicemonitors/ for Prometheus discovery of application metrics

  * namespace.yaml for monitoring namespace resources

  * a local README.md documenting the monitoring architecture

3. Bootstrap resources in bootstrap/

This directory contains manifests needed during the integration between EKS and GitOps, such as:

  * ALB controller service account bootstrap

  * aws-auth-related bootstrap manifests

4. RBAC resources in rbac/

This directory contains platform RBAC manifests such as:

  * application roles

  * monitoring roles

  * role bindings

---

## Relationship to the infrastructure repository

This repository assumes the EKS cluster already exists.

Infrastructure provisioning is handled separately in:

```text
eks-infrastructure
```

In short:

  * eks-infrastructure provisions the cluster

  * eks-gitops-apps configures what runs inside the cluster

---

## Observability in this repository

The monitoring model is split across two places:

  * apps/monitoring/ for the ArgoCD application definition and stack deployment

  * monitoring/ for supporting monitoring resources such as ServiceMonitors, alerts, and dashboards

This separation makes it easier to distinguish between:

  * deployment of the monitoring stack itself

  * monitoring content applied on top of that stack

---

## Example managed capabilities

Depending on the manifests currently enabled, this repository can manage:

  * Prometheus stack deployment

  * Alertmanager configuration

  * Grafana dashboards

  * ServiceMonitor resources for application metrics

  * application ingress and service definitions

  * ALB integration for workloads

---

## Operating model

Typical process:

### Update an application

Edit manifests under the relevant app directory, then push changes.

### Update monitoring resources

Edit files under monitoring/alerts, monitoring/dashboards, or monitoring/servicemonitors, then let ArgoCD sync them.

### Update access / permissions

Modify manifests under rbac/.

---

## Notes

This repository is the desired-state source for in-cluster resources.

Avoid making long-lived manual changes directly in the cluster.

If a resource is managed by ArgoCD, the Git state should be treated as authoritative.

---

## Author

Qays Alnajjad


















