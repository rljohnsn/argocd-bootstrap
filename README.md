# ArgoCD Bootstrap 

A strategy for managing the deployment of many microservices across many K8s clusters.

## ApplicationSets and Applications
Making use of [ApplicationSet](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/applicationset-specification/) ArgoCD objects we can provide a simple pattern to manage any number of service deployments via Helm.

In essence an [ApplicationSet](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/applicationset-specification/) is a template used to generate [Application](https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/) objects. Given a specific strategy for how to render them we can minimize the amount of metadata needing repetition. 

Using the [Matrix](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Matrix/) generator strategy gives the most flexibility with a minimal amount of metadata. It essentially combines two other ApplicationSet generator strategies into one. The two utilized in this example are:

* Git: files stored in a git repository for each ApplicationSet
* Clusters: the clusters registered with the running ArgoCD installation.

This works for one to one mapping of ArgoCD to K8s clusters and for many to many.

## Context

One new feature as of ArgoCD v2.6.x is the multiple `sources` feature for Application objects. This allows one to compose an Application definition from more than one resource. i.e. a Helm artifact and or one or more git repos containing values overrides.

## Proposed Structure

Using the described strategy we will break the components into three resources:

* Argo Application Set Git Repository
    * [argocd-bootstrap](https://github.com/rljohnsn/argocd-bootstrap): app set definitions and render strategies
* Argo Application Context Values
    * [argocd-env](https://github.com/rljohnsn/argocd-env): contextual values for environments / k8s clusters for each service
* Generic Helm Chart
    * [argocd-helm](https://github.com/rljohnsn/argocd-helm): sample chart

```bash
# Argo ApplicationSets
argocd-bootstrap
├── applications
|   | # Application metadata files
│   ├── foundation
│   │   ├── external-dns.yaml
│   │   └── k8s-dashboard.yaml
│   └── sample
│       ├── sample-service.yaml
│       ├── sample2-service.yaml
│       ├── sample3-service.yaml
│       └── sample4-service.yaml
└── manifests
    │ # Strategies for loading applications
    ├── appsets
    │   ├── foundation-appset.yaml
    │   └── sample-appset.yaml
    │ # Private git repo 
    └── repos
        └── docker-helm-repo.yaml

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
      # Prefix the ArgoCD Application CRD name with the cluster name
      # this gaurantee's uniqueness when dealing with multiple clusters
      # from a single instance of ArgoCD
      name: '{{name}}-{{component}}'
    spec:
      project: 'sample'
      sources:
        # Sample Helm Chart
        - repoURL: '{{chartURL}}'
          targetRevision: '{{chartVersion}}' 
          path: '{{chartPath}}'
          helm: 
            # Helm releaseName resets the name of the installation
            # in the target cluster back to the expected name vs.
            # the ArgoCD Application CRD name which prepends the cluster name
            releaseName: '{{component}}'
            ignoreMissingValueFiles: true
            valueFiles:
              - $values/base/values.yaml
              - $values/base/{{component}}-values.yaml
              - $values/env/{{metadata.labels.env}}/values.yaml
              - $values/env/{{metadata.labels.env}}/{{component}}-values.yaml
              - $values/variant/{{metadata.labels.region}}/values.yaml
              - $values/variant/{{metadata.labels.region}}/{{component}}-values.yaml
              - $values/variant/{{metadata.labels.region}}/{{name}}/values.yaml
              - $values/variant/{{metadata.labels.region}}/{{name}}/{{component}}-values.yaml
        # Values overrides, referenced above as $values
        - repoURL: '{{valuesURL}}'
          targetRevision: '{{valuesVersion}}' 
          path: '{{valuesPath}}'
          ref: values
      destination:
        server: '{{server}}'
        namespace: '{{namespace}}'
```

Some values in the ArgoCD [ApplicationSet](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/applicationset-specification/) template come from the [ApplicationSet](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/applicationset-specification/) [Generator](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/). Namely [`{{name}}`,`{{server}}`]

The clusters registered within ArgoCD may include labels and or annotations. Both of which may also be referenced as part of the `{{metdata}}` attribute provided by the ApplicationSet Generator.


```yaml
# Application Metadata read by the generator
component: sample
chart: argocd-helm
# Chart URL can be Helm Repository or Gitrepo
chartURL: https://github.com/rljohnsn/argocd-helm.git
# If using a helm repository, chartVersion would be chart
# If using a gitrepo branch and tags can be used
chartVersion: HEAD
chartPath: '.'
valuesURL: https://github.com/rljohnsn/argocd-env.git
valuesVersion: HEAD
valuesPath: '.'
namespace: sample
values: ''

```




We can expand the multiple sources and pull from distinct repos at each level.

```yaml
      sources:
        # Sample Helm Chart
        - repoURL: '{{chartURL}}'
          targetRevision: '{{chartVersion}}' 
          path: '{{chartPath}}'
          helm: 
            releaseName: '{{component}}'
            ignoreMissingValueFiles: true
            valueFiles:
              - $base/values.yaml
              - $env/{{metadata.labels.env}}/values.yaml
              - $env/{{metadata.labels.env}}/{{component}}-values.yaml
              - $variant/{{metadata.labels.region}}/values.yaml
              - $variant/{{metadata.labels.region}}/{{component}}-values.yaml
              - $variant/{{metadata.labels.region}}/{{name}}/values.yaml
        # Global values overrides
        - repoURL: '{{valuesURL}}'
          targetRevision: '{{valuesVersion}}' 
          path: 'base'
          ref: base
        # Service / Env specific overrides
        - repoURL: '{{valuesURL}}'
          targetRevision: '{{valuesVersion}}' 
          path: 'env'
          ref: env
        # Variants
        - repoURL: '{{valuesURL}}'
          targetRevision: '{{valuesVersion}}' 
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
          # https://argocd-applicationset.readthedocs.io/en/stable/Generators-Cluster/
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

> k -N argocd apply -f /manifests/appsets/*-appset.yaml
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
