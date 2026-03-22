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


## README مقترح لـ `eks-gitops-apps`

هذا المستودع هو مصدر الحقيقة المعلن للحالة داخل الكلاستر بعد البوتستراب: `apps/kustomization.yaml` يجمع ستة تطبيقات/مسارات، و`monitoring-chart/app.yaml` ينشر `kube-prometheus-stack` من chart خارجي مع values محلية، و`telegram-alerts` يملك `app.yaml` و`deployment.yaml` و`service.yaml` و`secret.yaml` عبر `kustomization.yaml`.  [oai_citation:3‡GitHub](https://github.com/QaysAlnajjad/eks-gitops-apps/blob/main/apps/kustomization.yaml)

كما أن `flask-app` يملك موارده الخاصة (`deployment.yaml` و`service.yaml` و`metrics-service.yaml` و`ingress.yaml`) و`sample-app` يملك `deployment.yaml` و`service.yaml` و`ingress.yaml` و`hpa.yaml` و`namespace.yaml`.  [oai_citation:4‡GitHub](https://github.com/QaysAlnajjad/eks-gitops-apps/tree/main/apps/flask-app)

```md
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
- Flask application
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
│   │   ├── app.yaml
│   │   ├── alerts/
│   │   ├── dashboards/
│   │   └── servicemonitors/
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
GitOps Model

The root application created from eks-infrastructure points ArgoCD to apps/.

ArgoCD then reconciles the application layer by reading apps/kustomization.yaml, which aggregates:
	•	aws-load-balancer-controller/app.yaml
	•	sample-app/app.yaml
	•	flask-app/app.yaml
	•	monitoring-chart/app.yaml
	•	monitoring-resources/app.yaml
	•	telegram-alerts/app.yaml

This means changes are applied through Git commits, not by manually patching the cluster.

⸻

Application Layout

1. AWS Load Balancer Controller

apps/aws-load-balancer-controller/

This application is deployed through ArgoCD from the external AWS EKS Helm charts repository and uses local values for:
	•	cluster name
	•	service account
	•	IRSA role annotation
	•	region
	•	VPC ID

⸻

2. Sample App

apps/sample-app/

A demo workload used as a simple application target, with:
	•	namespace manifest
	•	deployment
	•	service
	•	ingress
	•	HPA

This gives a separate example application distinct from the Flask application.

⸻

3. Flask App

apps/flask-app/

The Flask application is deployed from manifests stored in this repository. The directory includes:
	•	deployment
	•	service for user traffic
	•	separate metrics service
	•	ingress
	•	ArgoCD application definition

This separation allows user HTTP traffic and Prometheus metrics scraping to be handled independently.

⸻

4. Monitoring Chart

apps/monitoring-chart/

This application installs the monitoring stack using the kube-prometheus-stack Helm chart through ArgoCD.

The chart values are stored locally in:

apps/monitoring-chart/values.yaml

This is the base monitoring platform layer.

⸻

5. Monitoring Resources

apps/monitoring-resources/

This directory contains monitoring objects that sit on top of the monitoring chart, such as:
	•	PrometheusRule
	•	AlertmanagerConfig
	•	ServiceMonitor
	•	dashboards

This split is intentional:
	•	monitoring-chart installs the platform
	•	monitoring-resources customizes what that platform monitors and how it alerts

⸻

6. Telegram Alerts

apps/telegram-alerts/

This application deploys the webhook service that receives Alertmanager webhooks and forwards them to Telegram.

It is managed as its own ArgoCD application and includes:
	•	deployment
	•	service
	•	secret
	•	local kustomization

This makes the alert delivery path explicit and independently deployable.

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





















