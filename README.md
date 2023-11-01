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
# Argo ApplicationSets
argocd-bootstrap
├── README.md
│ # Strategy for loading foundation-charts
├── foundation-appset.yaml
├── foundation-charts
| # Application metadata file
│   ├── k8s-dashboard.yaml
│   └── ...
│ # Strategy for loading sample-charts
├── sample-appset.yaml
└── sample-charts
    | # Application metadata file
    ├── sample-service.yaml
    └── ...

# Application Values 
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

# Application Artifact (could also be a Helm repo)
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
# ApplicationSet Template
  template:
    metadata:
      name: '{{component}}'
    spec:
      project: 'sample'
      sources:
        # Sample Helm Chart
        - repoURL: '{{chartURL}}'
          targetRevision: HEAD 
          path: '.'
          helm: 
            releaseName: '{{component}}'
            ignoreMissingValueFiles: true
            valueFiles:
              - $values/base/values.yaml
              - $values/base/{{component}}-values.yaml
              - $values/env/{{env}}/values.yaml
              - $values/env/{{env}}/{{component}}-values.yaml
              - $values/variant/{{region}}/values.yaml
              - $values/variant/{{region}}/{{component}}-values.yaml
              - $values/variant/{{region}}/{{cluster}}/values.yaml
              - $values/variant/{{region}}/{{cluster}}/{{component}}-values.yaml
        # Values overrides, referenced above as $values
        - repoURL: '{{valuesURL}}'
          targetRevision: HEAD 
          path: '.'
          ref: values
      destination:
        server: '{{server}}'
        namespace: '{{namespace}}'
```

```yaml
# Application Metadata read by the generator
component: sample
# Chart URL can be Helm Repository or Gitrepo
chartURL: https://github.com/rljohnsn/argocd-helm.git
valuesURL: https://github.com/rljohnsn/argocd-env.git
version: 1.0.0
namespace: sample
```




We can expand the multiple sources and pull from distinct repos at each level.

```yaml
      sources:
        # Sample Helm Chart
        - repoURL: '{{chartURL}}'
          targetRevision: HEAD 
          path: '.'
          helm: 
            releaseName: '{{component}}'
            ignoreMissingValueFiles: true
            valueFiles:
              - $base/values.yaml
              - $env/{{env}}/values.yaml
              - $env/{{env}}/{{component}}-values.yaml
              - $variant/{{region}}/values.yaml
              - $variant/{{region}}/{{component}}-values.yaml
              - $variant/{{region}}/{{cluster}}/values.yaml
        # Global values overrides
        - repoURL: https://github.com/rljohnsn/argocd-env.git
          targetRevision: HEAD 
          path: 'base'
          ref: base
        # Service / Env specific overrides
        - repoURL: https://github.com/rljohnsn/argocd-env.git
          targetRevision: HEAD 
          path: 'env'
          ref: env
        # Variants
        - repoURL: https://github.com/rljohnsn/argocd-env.git
          targetRevision: HEAD 
          path: 'variant'
          ref: variant


```

## ApplicationSet Matrix Generator

The Matrix generator may be used to combine the generated parameters of two separate generators.

```yaml
spec:
  generators:
    # matrix 'parent' generator
    - matrix:
        generators:
          # git generator, 'child' #1
          - git:
              repoURL: https://github.com/rljohnsn/argocd-bootstrap.git
              revision: main
              files:
                - path: "foundation-charts/"
          # cluster generator, 'child' #2
          # {} matches any and all clusters registered
          - clusters: {}

```

## Why?

Adhering to a [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) strategy we want to minimize the amount of repeated metadata. 

Using an ApplicationSet, an Application with multiple sources allows us to render any number of applications against any number of clusters with minimal to no repeated settings/metadata.


## How do I use this repo?

```bash
# pre-requisites 
> kubectl cluster-info
Kubernetes control plane is running at https://0.0.0.0:51527
CoreDNS is running at https://0.0.0.0:51527/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

❯ argocd cluster list
SERVER                          NAME        VERSION  STATUS      MESSAGE  PROJECT
https://kubernetes.default.svc  in-cluster  1.26     Successful


# fork the sample repos
https://github.com/rljohnsn/argocd-bootstrap
https://github.com/rljohnsn/argocd-env
https://github.com/rljohnsn/argocd-helm 

# Clone the bootstrap local
> git clone argocd-bootstrap

# Update appropriate repoURL's in appset and app metadata files to reflect your forked repos and commit back to the repo
argocd-bootstrap>git push origin main
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Delta compression using up to 10 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 512 bytes | 512.00 KiB/s, done.
Total 3 (delta 1), reused 0 (delta 0), pack-reused 0
remote: Resolving deltas: 100% (1/1), completed with 1 local object.
To github.com:rljohnsn/argocd-bootstrap.git
   7f7bd5d..2143410  main -> main

> k -N argocd apply -f *-appset.yaml
appproject.argoproj.io/foundation created
applicationset.argoproj.io/foundation-appset created
appproject.argoproj.io/sample created
applicationset.argoproj.io/sample-appset created

# check your argocd UI

> argocd proj list
NAME        DESCRIPTION                                                                   DESTINATIONS  SOURCES  CLUSTER-RESOURCE-WHITELIST  NAMESPACE-RESOURCE-BLACKLIST  SIGNATURE-KEYS  ORPHANED-RESOURCES
default                                                                                   *,*           *        */*                         <none>                        <none>          disabled
sample      Sample set of services                                                        *,sample      *        /Namespace                  <none>                        <none>          disabled
foundation  Supporting services to make a K8s cluster ready to host application services  *,*           *        */*                         <none>                        <none>          disabled


```