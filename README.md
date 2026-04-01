# eks-gitops-apps

GitOps repository for the Kubernetes application layer running on the EKS cluster.

This repository is the declarative source of truth for in-cluster applications and supporting resources managed by ArgoCD after the infrastructure bootstrap is complete.

---

## Table of Contents 

- [Overview](#overview)
- [What This Repository Owns](#what-this-repository-owns)
- [Repository Structure](#repository-structure)
- [GitOps Model](#gitops-model)
- [Production Thinking: GitOps Validation & Safety](#production-thinking-gitops-validation--safety)
- [Application Layout](#application-layout)
- [Monitoring Architecture](#monitoring-architecture)
- [Why the Monitoring Split Matters](#why-the-monitoring-split-matters)
- [Deployment Flow](#deployment-flow)
- [Operating Model](#operating-model)
- [Notes](#notes)
- [Relationship to the infrastructure repository](#relationship-to-the-infrastructure-repository)
- [Author](#author)


## Overview

This repository manages the Kubernetes layer after `eks-infrastructure` has:

- provisioned the EKS cluster
- installed ArgoCD
- created the root ArgoCD application

From that point onward, ArgoCD continuously reconciles the content of this repository to the cluster.

---

## What This Repository Owns

This repository owns the **application and platform layer inside Kubernetes**, including:

- AWS Load Balancer Controller application
- sample application
- flask application
- monitoring chart deployment
- monitoring resources layered on top of the monitoring stack
- Telegram alert webhook application

This repository is intentionally separate from the Terraform repository so infrastructure provisioning and Kubernetes desired state remain cleanly separated.

---

## Repository Structure

```text
eks-gitops-apps/
├── apps/
│   ├── aws-load-balancer-controller/
│   │   ├── app.yaml
│   │   └── values.yaml
│   ├── flask-app/
│   │   ├── app.yaml
│   │   ├── deployment.yaml
│   │   ├── ingress.yaml
│   │   ├── metrics-service.yaml
│   │   └── service.yaml
│   ├── monitoring-chart/
│   │   ├── app.yaml
│   │   └── values.yaml
│   ├── monitoring-resources/
│   │   ├── servicemonitors/
│   │   ├── alerts/
|   |   |   ├── alertmanager-config.yaml
|   |   |   └── app-alerts.yaml
│   │   ├── dashboards/
|   |   |   └── flask-dashboard.json
|   |   ├── servicemonitors/
|   |   |   └── flask.yaml
|   |   ├── app.yaml
│   │   └── kustomization.yaml
│   ├── sample-app/
│   │   ├── app.yaml
│   │   ├── deployment.yaml
│   │   ├── hpa.yaml
│   │   ├── ingress.yaml
│   │   ├── namespace.yaml
│   │   └── service.yaml
│   ├── telegram-alerts/
│   │   ├── app.yaml
│   │   ├── deployment.yaml
│   │   ├── kustomization.yaml
│   │   ├── secret.yaml
│   │   └── service.yaml
│   └── kustomization.yaml
└── README.md
```

---

## GitOps Model

The root application created from eks-infrastructure points ArgoCD to apps/.

ArgoCD then reconciles the application layer by reading apps/kustomization.yaml, which aggregates:

  * aws-load-balancer-controller/app.yaml
  * sample-app/app.yaml
  * flask-app/app.yaml
  * monitoring-chart/app.yaml
  * monitoring-resources/app.yaml
  * telegram-alerts/app.yaml

This means changes are applied through Git commits, not by manually patching the cluster.

Workflow:

  1. Change manifests in Git
  2. Push to repository
  3. ArgoCD detects drift
  4. ArgoCD syncs the cluster to the desired state

This keeps deployments declarative, traceable, and auditable.

---

## Production Thinking: GitOps Validation & Safety

This repository follows a GitOps approach where Kubernetes manifests are managed declaratively and applied via ArgoCD.

To ensure safety and correctness before changes reach the cluster, a validation pipeline is implemented as part of CI.

### Validation Workflow

The CI pipeline performs two main steps:

#### 1. Render (Kustomize)

All `kustomization.yaml` files under the `apps/` directory are rendered using:

```
kubectl kustomize
```

This step converts overlays and compositions into fully expanded Kubernetes manifests.

Each kustomization is rendered into a separate file under the `rendered/` directory.

#### 2. Schema Validation (kubeconform)

The rendered manifests are then validated using:

```
kubeconform
```

This ensures that:

* manifests conform to Kubernetes API schemas
* invalid configurations are caught early
* broken deployments are prevented before reaching the cluster

---

### Why Rendering Before Validation Matters

Validating raw manifests is not sufficient in a GitOps setup using Kustomize.

Issues often appear only after rendering, such as:

* incorrect resource composition
* missing fields after overlays
* invalid final manifests

By validating the rendered output, the pipeline checks what will actually be applied to the cluster.

---

### Handling Custom Resources (CRDs)

The validation step uses:

```
-ignore-missing-schemas
```

This is intentional.

Many Kubernetes platforms rely on Custom Resource Definitions (CRDs), such as:

* Prometheus Operator (`ServiceMonitor`, `Alertmanager`)
* AWS Load Balancer Controller resources
* other operator-managed resources

These resources do not always have publicly available JSON schemas.

Without this flag, validation would fail on valid CRDs simply because schemas are not available.

---

### Engineering Trade-off

Using `-ignore-missing-schemas` reflects a real-world trade-off:

* ✔️ strict validation for core Kubernetes resources
* ✔️ flexibility for CRDs and platform extensions
* ❌ not all resources can be fully schema-validated

This approach ensures the pipeline remains:

* practical
* reliable
* aligned with production environments

---

### Outcome

This validation workflow ensures that:

* only valid Kubernetes manifests are committed
* configuration errors are detected early
* GitOps deployments remain safe and predictable

It mirrors production-grade platform practices where validation happens before reconciliation, not after failure.

---

## Application Layout

### 1. AWS Load Balancer Controller

apps/aws-load-balancer-controller/

This application is deployed through ArgoCD from the external AWS EKS Helm charts repository and uses local values for:

  * cluster name
  * service account
  * IRSA role annotation
  * region
  * VPC ID

Before syncing this application, update `apps/aws-load-balancer-controller/values.yaml` with environment-specific values from the infrastructure repository, including:

- EKS cluster name
- AWS region
- VPC ID
- IRSA role ARN / AWS account ID

Example fields:

```yaml
clusterName: eks-cluster

serviceAccount:
  create: true
  name: aws-load-balancer-controller
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<AWS_ACCOUNT_ID>:role/eks-alb-controller-role

region: us-east-1

vpcId: <VPC_ID>
```

---

### 2. Sample App

apps/sample-app/

A demo workload used as a simple application target, with:

  *	namespace manifest
  *	deployment
  *	service
  *	ingress
  *	HPA

This gives a separate example application distinct from the Flask application.

---

### 3. Flask App

apps/flask-app/

The Flask application is deployed from manifests stored in this repository. The directory includes:

  *	deployment
  *	service for user traffic
  *	separate metrics service
  *	ingress
  *	ArgoCD application definition

This separation allows user HTTP traffic and Prometheus metrics scraping to be handled independently.

---

### 4. Monitoring Chart

apps/monitoring-chart/

This application installs the monitoring stack using the kube-prometheus-stack Helm chart through ArgoCD.

The chart values are stored locally in:

  * apps/monitoring-chart/values.yaml

This is the base monitoring platform layer.

---

### 5. Monitoring Resources

apps/monitoring-resources/

This directory contains monitoring objects that sit on top of the monitoring chart, such as:

  *	PrometheusRule
  *	AlertmanagerConfig
  *	ServiceMonitor
  *	dashboards

This split is intentional:

  *	monitoring-chart installs the platform
  *	monitoring-resources customizes what that platform monitors and how it alerts

---

### 6. Telegram Alerts

apps/telegram-alerts/

This application deploys the webhook service that receives Alertmanager webhooks and forwards them to Telegram.

It is managed as its own ArgoCD application and includes:

  *	deployment
  *	service
  *	secret
  *	local kustomization

This makes the alert delivery path explicit and independently deployable.

---

## Monitoring Architecture

The monitoring flow in this repository is:

Flask metrics endpoint
  -> ServiceMonitor
  -> Prometheus
  -> PrometheusRule
  -> Alertmanager
  -> telegram-alerts webhook
  -> Telegram

This design separates:

  *	platform installation
  *	monitoring configuration
  *	notification delivery

into clear GitOps-managed components.

---

## Why the Monitoring Split Matters

The repository uses two monitoring-related applications for a reason:

### monitoring-chart

Owns installation of the Prometheus / Alertmanager / Grafana stack itself.

### monitoring-resources

Owns project-specific monitoring objects such as:

  *	alert rules
  *	service discovery
  *	dashboards
  *	Alertmanager routing config

This keeps the Helm chart clean while preserving flexibility for project-specific observability resources.

---

## Deployment Flow

  1. eks-infrastructure creates the cluster and installs ArgoCD
  2. bootstrap/root-app.yaml points ArgoCD to this repository
  3. ArgoCD reads apps/kustomization.yaml
  4. Each application in apps/ is reconciled into the cluster
  5. Ongoing changes are delivered by commit + sync
  6. Create required Kubernetes secrets manually (not stored in Git)

      Sensitive data (e.g., API tokens) is not stored in this repository.
      Required secrets must be created in the cluster before applications depending on them are deployed.
---

## Operating Model

Make changes by editing Git manifests, then push.

### Secret Handling

Secrets are not stored in this repository.

Before deploying applications that depend on sensitive data (such as Telegram alerts), required Kubernetes secrets must be created manually in the cluster.

Example:

kubectl create secret generic telegram-webhook-secret \
  -n monitoring \
  --from-literal=TELEGRAM_BOT_TOKEN=$TELEGRAM_BOT_TOKEN \
  --from-literal=TELEGRAM_CHAT_ID=$TELEGRAM_CHAT_ID

These secrets are referenced by the application manifests but are intentionally excluded from Git to prevent credential exposure.

### Typical workflow

git add .
git commit -m "update application manifests"
git push

ArgoCD will detect drift and reconcile the cluster to match this repository.

---

## Notes

  *	This repository should remain the single source of truth for in-cluster desired state
  *	Avoid manual kubectl edit changes unless debugging temporarily
  *	Permanent changes should always be committed back to Git

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

## Author

Qays Alnajjad




