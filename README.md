# Helm + ArgoCD Multi-Stage Deployment

## Overview
This repository demonstrates how to manage **multi-stage deployments** (like `dev`, `test`, `prod`) using **Helm** and **ArgoCD**. The goal is to dynamically create ArgoCD `Application` manifests based on multiple environments and applications using Helm templating.

---

## Repository Structure
```bash
helm-chart/
├── Chart.yaml              # Helm chart definition
├── stages.yaml             # List of stages/environments
├── values-dev.yaml         # Dev-specific application values
├── values-test.yaml        # Test-specific application values
└── templates/
    └── application.yaml    # Template for ArgoCD Application CRDs
```

---

## `stages.yaml`
Defines the stages or environments and corresponding Kubernetes namespaces:
```yaml
stages:
  - name: dev
    namespace: dev
  - name: test
    namespace: test
```

---

## `values-dev.yaml` / `values-test.yaml`
Contains the application metadata and image information for a specific stage.

### Example `values-dev.yaml`
```yaml
applications:
  - name: springboot-hello-world
    image:
      tag: "20250424.093922.0"
```

### Example `values-test.yaml`
```yaml
applications:
  - name: springboot-hello-world
    image:
      tag: "20250424.3.0"
```

---

## `application.yaml` Template
Generates ArgoCD Applications for each stage and application.

```gotemplate
{{- range $stage := .Values.stages }}
  {{- range $app := $.Values.applications }}
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: {{ $stage.name }}-{{ $app.name }}
  namespace: argocd
spec:
  project: default
  source:
    repoURL: registry-1.docker.io/devstackhub
    chart: dox-cli
    targetRevision: 1.0.229-helm
    helm:
      parameters:
        - name: image.repository
          value: devstackhub/{{ $app.name }}
        - name: service.name
          value: {{ $app.name }}
        - name: global.stage
          value: {{ $stage.name }}
      values: |
        {{ . | toYaml | nindent 8 }}
  destination:
    server: https://kubernetes.default.svc
    namespace: {{ $stage.namespace }}
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
  {{- end }}
{{- end }}
```

---

## ArgoCD Bootstrap Configuration
This approach uses a single ArgoCD `Application` to bootstrap all the generated `Application` manifests using Helm.

### `argo-apps-bootstrap.yaml`
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-apps-bootstrap
  namespace: argocd
spec:
  destination:
    namespace: argocd
    server: https://kubernetes.default.svc
  source:
    path: argo-apps
    repoURL: https://github.com/dopxlab/gitops-argocd.git
    targetRevision: HEAD
    helm:
      valueFiles:
        - stages.yaml
        - values-dev.yaml
        - values-test.yaml
  sources: []
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

> This will automatically render and deploy all ArgoCD `Application` CRs into your cluster.

---

## Using `helm template`
To generate the manifests locally:

### For Dev Stage:
```bash
helm template my-release . -f stages.yaml -f values-dev.yaml
```

### For Test Stage:
```bash
helm template my-release . -f stages.yaml -f values-test.yaml
```

> This will render separate ArgoCD Applications based on stage and application names.

---

## How ArgoCD Works with This Chart
- ArgoCD syncs the generated `Application` manifests into your cluster.
- Each application is deployed into the stage's defined namespace.
- `syncPolicy` is set to `automated`, so ArgoCD will auto-deploy and self-heal.

---

## Best Practices
- Always include `stages.yaml` when rendering manifests.
- Separate environment-specific values in `values-{stage}.yaml`.
- Use a CI pipeline to automate manifest rendering and GitOps-based delivery.

---

## Summary
| Component         | Description                                      |
|-------------------|--------------------------------------------------|
| `stages.yaml`     | Defines all deployment stages                    |
| `values-dev.yaml` | Dev stage application values                     |
| `application.yaml`| Helm template for ArgoCD Applications            |
| `argo-apps-bootstrap` | Central Application to bootstrap stage apps |
| ArgoCD            | Deploys, syncs, and monitors each Application    |

---

## License
MIT

## Maintainer
**Joby Pooppillikudiyil / DoX CLI Team**

