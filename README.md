# eks-gitops-apps

GitOps repository for the Kubernetes application layer running on the EKS cluster.

This repository is the declarative source of truth for in-cluster applications and supporting resources managed by ArgoCD after the infrastructure bootstrap is complete.

---

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

## Application Layout

### 1. AWS Load Balancer Controller

apps/aws-load-balancer-controller/

This application is deployed through ArgoCD from the external AWS EKS Helm charts repository and uses local values for:

  * cluster name
  * service account
  * IRSA role annotation
  * region
  * VPC ID

⸻

### 2. Sample App

apps/sample-app/

A demo workload used as a simple application target, with:

  *	namespace manifest
  *	deployment
  *	service
  *	ingress
  *	HPA

This gives a separate example application distinct from the Flask application.

⸻

### 3. Flask App

apps/flask-app/

The Flask application is deployed from manifests stored in this repository. The directory includes:

  *	deployment
  *	service for user traffic
  *	separate metrics service
  *	ingress
  *	ArgoCD application definition

This separation allows user HTTP traffic and Prometheus metrics scraping to be handled independently.

⸻

### 4. Monitoring Chart

apps/monitoring-chart/

This application installs the monitoring stack using the kube-prometheus-stack Helm chart through ArgoCD.

The chart values are stored locally in:

  * apps/monitoring-chart/values.yaml

This is the base monitoring platform layer.

⸻

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

⸻

### 6. Telegram Alerts

apps/telegram-alerts/

This application deploys the webhook service that receives Alertmanager webhooks and forwards them to Telegram.

It is managed as its own ArgoCD application and includes:
	•	deployment
	•	service
	•	secret
	•	local kustomization

This makes the alert delivery path explicit and independently deployable.


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



































⸻

Monitoring Architecture

The monitoring flow in this repository is:

Flask metrics endpoint
  -> ServiceMonitor
  -> Prometheus
  -> PrometheusRule
  -> Alertmanager
  -> telegram-alerts webhook
  -> Telegram

This design separates:
	•	platform installation
	•	monitoring configuration
	•	notification delivery

into clear GitOps-managed components.

⸻

Why the Monitoring Split Matters

The repository uses two monitoring-related applications for a reason:

monitoring-chart

Owns installation of the Prometheus / Alertmanager / Grafana stack itself.

monitoring-resources

Owns project-specific monitoring objects such as:
	•	alert rules
	•	service discovery
	•	dashboards
	•	Alertmanager routing config

This keeps the Helm chart clean while preserving flexibility for project-specific observability resources.

⸻

Deployment Flow
	1.	eks-infrastructure creates the cluster and installs ArgoCD
	2.	bootstrap/root-app.yaml points ArgoCD to this repository
	3.	ArgoCD reads apps/kustomization.yaml
	4.	Each application in apps/ is reconciled into the cluster
	5.	Ongoing changes are delivered by commit + sync

⸻

Operating Model

Make changes by editing Git manifests, then push.

Typical workflow:

git add .
git commit -m "update application manifests"
git push

ArgoCD will detect drift and reconcile the cluster to match this repository.

⸻

Notes
	•	This repository should remain the single source of truth for in-cluster desired state
	•	Avoid manual kubectl edit changes unless debugging temporarily
	•	Permanent changes should always be committed back to Git

⸻

Related Repository
	•	eks-infrastructure: provisions AWS/EKS and bootstraps ArgoCD

إذا أردت، أرتب لك الآن **نسخة نهائية مختصرة وجاهزة للنسخ مباشرة** إلى `README.md` في كل repo، أو نسخة **أكثر احترافية مع قسم Architecture ورسوم ASCII**.





















