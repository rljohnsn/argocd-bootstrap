## Application Metadata

Each sub directory here is a grouping of application metadata referenced by an [ApplicationSet](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/applicationset-specification/) object in [manifests](manifests) directory.

An [ApplicationSet](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/applicationset-specification/) template has certain values it references from these metadata files to generate [Application](https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/) objects for each.

```yaml
component: kubernetes-dashboard
chart: kubernetes-dashboard
chartURL: https://kubernetes.github.io/dashboard
chartVersion: 5.11.0
chartPath: '.'
valuesURL: https://github.com/rljohnsn/argocd-env.git
valuesVersion: HEAD
valuesPath: '.'
namespace: kubernetes-dashboard
values: |-
  fullnameOverride: "kubernetes-dashboard"
  nameOverride: "kubernetes-dashboard"
```

The keys above [`component`,`repoURL`, etc.] are referenced in the [ApplicationSet](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/applicationset-specification/) as you can see below:

```yaml
  template:
    metadata:
      # Prefix the ArgoCD Application CRD name with the cluster name
      # this gaurantee's uniqueness when dealing with multiple clusters
      # from a single instance of ArgoCD
      name: '{{name}}-{{component}}'
    spec:
      project: 'foundation'
      sources:
        # Sample Helm Chart
        - repoURL: '{{chartURL}}'
          chart: '{{chart}}'
          targetRevision: '{{chartVersion}}' 
          path: '{{chartPath}}'
          helm: 
            # Helm releaseName resets the name of the installation
            # in the target cluster back to the expected name vs.
            # the ArgoCD Application CRD name which prepends the cluster name
            releaseName: '{{component}}'
            values: |-
              {{values}}
            ignoreMissingValueFiles: true
            valueFiles:
              - '$values/base/values.yaml'
              - '$values/base/{{component}}-values.yaml'
              - '$values/env/{{metadata.labels.env}}/values.yaml'
              - '$values/env/{{metadata.labels.env}}/{{component}}-values.yaml'
              - '$values/variant/{{metadata.labels.region}}/values.yaml'
              - '$values/variant/{{metadata.labels.region}}/{{component}}-values.yaml'
              - '$values/variant/{{metadata.labels.region}}/{{name}}/values.yaml'
              - '$values/variant/{{metadata.labels.region}}/{{name}}/{{component}}-values.yaml'
        # Values overrides
        - repoURL: '{{valuesURL}}'
          targetRevision: '{{valuesVersion}}' 
          path: '{{valuesPath}}'
          ref: values
      destination:
        server: '{{server}}'
        namespace: '{{namespace}}'

```

Some values in the ArgoCD [ApplicationSet](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/applicationset-specification/) template come from the [ApplicationSet](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/applicationset-specification/) [Generator](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/). Namely [`{{name}}`,`{{server}}`]