# ArgoCD Bootstrap 

A strategy for managing the deployment of many microservices across many K8s clusters.

## ApplicationSets and Applications
Making use of [ApplicationSet](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/applicationset-specification/) ArgoCD objects we can provide a simple pattern to manage any number of service deployments via Helm.

In essence an [ApplicationSet](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/applicationset-specification/) is a template used to generate [Application](https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/) objects. Given a specific strategy for how to render them we can minimize the amount of metadata needing repition. 

Using the [Matrix](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Matrix/) generator strategy gives the most flexibility with a minimal amount of metadata. It essentially combines two other ApplicationSet generator strategies into one. The two utilized in this example are:

* Git: files stored in a git repository for each ApplicationSet
* Clusters: the clusters registered with the running ArgoCD installation.

This works for one to one mapping of ArgoCD to K8s clusters and for many to many.

## Context

One new feature as of ArgoCD v2.6.x is the multiple `sources` feature for Application objects. This allows one to compose an Application definition from more than one resource. i.e. a Helm artifact and or one or more git repos containing values overrides.

## Proposed Structure

Using the described strategy we will break the components into three resources:

* Argo Application Set Git Repository
    * `argocd-bootstrap`: app set definitions and render strategies
* Argo Application Context Values
    * `argocd-env`: contextual values for environments / k8s clusters for each service
* Generic Helm Chart
    * `argocd-helm`: sample chart

```bash
argocd-bootstrap
├── README.md
│ # Strategy for loading foundation-charts
├── foundation-appset.yaml
├── foundation-charts
│   ├── external-secrets.yaml
│   ├── metrics-server.yaml
│   ├── reloader.yaml
│   └── ...
│ # Strategy for loading from sample-charts
├── sample-appset.yaml
└── sample-charts
    ├── sample-service.yaml
    └── ...

argocd-env
├── README.md
├── base
├── env
│   ├── beta
│   ├── preview
│   ├── prod
│   └── stage
└── variant
    ├── eu
    └── us

argocd-helm
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
└── values.yaml

```

## Application Multiple Sources
ArgoCD Application objects can take advantage of sourcing configuration, and or settings from more than one resource.

```yaml
sources:
  # Standard Postman Helm Chart
  - chart: argocd-helm-sample
    repoURL: https://github.com/rljohnsn/argocd-helm-sample.git
    targetRevision: HEAD 
    helm: 
      releaseName: '{{$.Values.sample.appName}}'
  # Global values overrides
  - repoURL: https://github.com/rljohnsn/argocd-env.git
    targetRevision: HEAD 
    path: 'base'
  # Service / Env specific overrides
  - repoURL: https://github.com/rljohnsn/argocd-env.git
    targetRevision: HEAD 
    path: '{{$.Values.sample.env}}/{{$.Values.sample.appName}}-values.yaml'


```

## ApplicationSet Matrix Generator

The Matrix generator may be used to combine the generated parameters of two separate generators.

```yaml

```

## Why?

Adhering to a [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) strategy we want to minimize the amount of repeated metadata. 

Using an ApplicationSet, an Application with multiple sources allows us to render any number of applications against any number of clusters with minimal to no repeated settings/metadata.
