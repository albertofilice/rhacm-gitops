# RedHat Advanced Cluster Manager - Openshift Gitops
\
\
\
![image](https://github.com/albertofilice/rhacm-gitops/assets/35273403/40839582-be3c-49ee-ab5b-8dfd12cf9d74)
\
\
\
These configurations allow a vanilla helm to be used fully automatically in RHACM and/or Gitops without modifying basic helm charts in any way. We are going to see how deployment of Helm-based RHACM applications with external value files or advanced Helm customizations works.

Here are the contents of this article:

A. Installing RedHat Advanced Cluster Manager (RHACM)

B. Installing Openshift Gitops (ArgoCD) on a Hub Cluster

C. Integrating Openshift Gitops with Red Advanced Cluster Manager

D. Define and create an ApplicationSet for deploying apps.

**DISCLAIMER**: This is only a means to show a possible helm-based application implementation on multiple clusters with RHACM and Gitops. It is not a supported guide but simply shows the necessary structures and methods. Do not use this model in a production environment without proper adaptation and testing.

**USEFUL ADDITIONS**
For your convenience, you can perform openshift-gitops and Red Hat Advanced Cluster Management installations by running the following commands:

```bash
git clone https://github.com/albertofilice/rhacm-gitops.git
cd rhacm-gitops
oc apply -k bootstrap/
```

In this example, we tested on the local cluster of Red Hat Advanced Cluster Manager, but everything is set up to deploy to multiple clusters. You need only edit the placement. For Examaple:

From
```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: placement-gitops-wordpress-multiple-sources
  namespace: openshift-gitops
spec:
  clusterSets:
    - sample-clusterset
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: local-cluster
              operator: In
              values:
                - "true"
```
to

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: placement-gitops-wordpress-multiple-sources
  namespace: openshift-gitops
spec:
  clusterSets:
    - sample-clusterset
```

# Installing Openshift-Gitops and RHACM

1. Create Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  annotations: {}
  name: openshift-gitops
spec: {}
---
apiVersion: v1
kind: Namespace
metadata:
  name: open-cluster-management
```
2. Operator Deployment

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: gitops-1.8
  installPlanApproval: Automatic
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: open-cluster-management
  namespace: open-cluster-management
spec:
  targetNamespaces:
  - open-cluster-management  
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: advanced-cluster-management
  namespace: open-cluster-management
spec:
  channel: release-2.7
  installPlanApproval: Automatic
  name: advanced-cluster-management
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

3. RBAC configuration

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: argocd-rbac-ca
subjects:
  - kind: ServiceAccount
    name: openshift-gitops-argocd-application-controller
    namespace: openshift-gitops
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
```

4. ArgoCD and RHACM Cluster Installation

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: openshift-gitops
  namespace: openshift-gitops
spec:
  server:
    autoscale:
      enabled: false
    grpc:
      ingress:
        enabled: false
    ingress:
      enabled: false
    resources:
      limits:
        cpu: 500m
        memory: 256Mi
      requests:
        cpu: 125m
        memory: 128Mi
    route:
      enabled: true
      tls:
        insecureEdgeTerminationPolicy: Redirect
        termination: reencrypt
    service:
      type: ''
  grafana:
    enabled: false
    ingress:
      enabled: false
    resources:
      limits:
        cpu: 500m
        memory: 256Mi
      requests:
        cpu: 250m
        memory: 128Mi
    route:
      enabled: false
  monitoring:
    enabled: false
  notifications:
    enabled: false
  prometheus:
    enabled: false
    ingress:
      enabled: false
    route:
      enabled: false
  initialSSHKnownHosts: {}
  sso:
    dex:
      openShiftOAuth: true
      resources:
        limits:
          cpu: 500m
          memory: 256Mi
        requests:
          cpu: 250m
          memory: 128Mi
    provider: dex
  applicationSet:
    resources:
      limits:
        cpu: '2'
        memory: 1Gi
      requests:
        cpu: 250m
        memory: 512Mi
    webhookServer:
      ingress:
        enabled: false
      route:
        enabled: false
  rbac:
    policy: |
      g, system:cluster-admins, role:admin
      g, cluster-admins, role:admin
    scopes: '[groups]'
  repo:
    resources:
      limits:
        cpu: '1'
        memory: 1Gi
      requests:
        cpu: 250m
        memory: 256Mi
  resourceExclusions: |
    - apiGroups:
      - tekton.dev
      clusters:
      - '*'
      kinds:
      - TaskRun
      - PipelineRun
  ha:
    enabled: false
    resources:
      limits:
        cpu: 500m
        memory: 256Mi
      requests:
        cpu: 250m
        memory: 128Mi
  kustomizeBuildOptions: '--enable-helm'
  tls:
    ca: {}
  redis:
    resources:
      limits:
        cpu: 500m
        memory: 256Mi
      requests:
        cpu: 250m
        memory: 128Mi
  controller:
    processors: {}
    resources:
      limits:
        cpu: '2'
        memory: 2Gi
      requests:
        cpu: 250m
        memory: 1Gi
    sharding: {}
---
apiVersion: operator.open-cluster-management.io/v1
kind: MultiClusterHub
metadata:
  name: multiclusterhub
  namespace: open-cluster-management
spec: {}    
```

5. RHACM RBAC

```bash
oc adm policy add-cluster-role-to-user --rolebinding-name=open-cluster-management:subscription-admin open-cluster-management:subscription-admin kube:admin
oc adm policy add-cluster-role-to-user --rolebinding-name=open-cluster-management:subscription-admin open-cluster-management:subscription-admin system:admin
```

6. Define ClusterSet

From RHACM Console

<img width="1506" alt="image" src="https://github.com/albertofilice/rhacm-gitops/assets/35273403/05ca337c-6eac-4d2b-a218-e271088ff5d0">


```bash
oc label managedcluster local-cluster cluster.open-cluster-management.io/clusterset=sample-clusterset --overwrite
```


# Integrating Openshift Gitops with Red Advanced Cluster Manager

ManagedClusterSetBinding and placement are used to intercept and configure the clusters present on RHACM and in ArgoCD. It is possible to customize these configurations by excluding specific clusters.

```yaml
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: ManagedClusterSetBinding
metadata:
  name: sample-clusterset
  namespace: openshift-gitops
spec:
  clusterSet: sample-clusterset
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: gitops-clusters
  namespace: openshift-gitops
spec:
  predicates:
  - requiredClusterSelector:
      labelSelector:
        matchExpressions:
        - key: vendor
          operator: "In"
          values:
          - OpenShift
---
apiVersion: apps.open-cluster-management.io/v1beta1
kind: GitOpsCluster
metadata:
  name: gitops-clusters
  namespace: openshift-gitops
spec:
  argoServer:
    cluster: local-cluster
    argoNamespace: openshift-gitops
  placementRef:
    kind: Placement
    apiVersion: cluster.open-cluster-management.io/v1beta1
    name: gitops-clusters
    namespace: openshift-gitops
```

# ApplicationSet creation and application management using Gitops

Three independent solutions were produced for application management using Gitops; we used the "wordpress" chart from https://charts.bitnami.com/bitnami as a demo.

**1. ApplicationSet with multiple sources**

The first solution involves implementing an ApplicationSet in Argocd that leverages the WordPress chart and instead pulls the helm value from an external git. This solution has as a prerequisite that the git repository in Argocd be configured (Only for private repositories), and there must be a single values.yaml in the repo and a path indicated in the ApplicationSet.

<img width="1512" alt="image" src="https://github.com/albertofilice/rhacm-gitops/assets/35273403/53d031e5-13b3-4aa4-9ae3-cfdecf768901">


```yaml
          - git:
              files:
                - path: example/multiple_sources/{{name}}/wordpress/values.yaml
              repoURL: 'https://github.com/albertofilice/rhacm-gitops.git'
              revision: main
```

Below is an example of ApplicationSet used for this case study:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: wordpress-multiple-sources
  namespace: openshift-gitops
spec:
  generators:
    - matrix:
        generators:
          # definition and interception of clusters via placemente RHACM.
          - clusterDecisionResource:
              configMapRef: acm-placement
              labelSelector:
                matchLabels:
                  cluster.open-cluster-management.io/placement: placement-gitops-wordpress-multiple-sources
            requeueAfterSeconds: 180
          # Definition of the external git containing the Value File. This file is read and interpreted as a single string, which can be used in the application creation template with the variable {{ values }}
          - git:
              files:
                - path: example/multiple_sources/{{name}}/wordpress/values.yaml
              repoURL: 'https://github.com/albertofilice/rhacm-gitops.git'
              revision: main
  template:
    metadata:
      labels:
        velero.io/exclude-from-backup: 'true'
      # name is autogenerated and contains the cluster name present in RHACM "{{name}}" in order to avoid duplicate applications. For example, "wordpress-multiple-sources-local-cluster"
      name: 'wordpress-multiple-sources-{{name}}'
    spec:
      # Targer of the installation, the server is a system variable and provides the API url, for example, "https://api.cluster.example.com:6443"
      destination:
        namespace: wordpress-multiple-sources
        server: '{{server}}'
      project: default
      source:
        chart: wordpress
        helm:
          # Definizione dei value helm e ovveride
          values: |
            {{values}}        
        repoURL: 'https://charts.bitnami.com/bitnami'
        targetRevision: 16.0.4
      # Defining sync options related to ArgoCD: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - PruneLast=true    
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: placement-gitops-wordpress-multiple-sources
  namespace: openshift-gitops
spec:
  clusterSets:
    - sample-clusterset
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: local-cluster
              operator: In
              values:
                - "true"

```

Example of values.yaml\
For this to work, all overrides must be contained in
`values: |`

```yaml
values: |
    image:
      registry: docker.io
      repository: bitnami/wordpress
      tag: 6.2.0-debian-11-r22
      digest: ""
      pullPolicy: IfNotPresent
      pullSecrets: []
      debug: false
    multisite:
      enable: false
      host: ""
      networkType: subdomain
      enableNipIoRedirect: false
```
# Final result

The end result of these implementations is the creation of an Argocd application, derived from the ApplicationSet and dynamically created based on the reference cluster configured in RedHat Advanced Cluster Manager.

<img width="1512" alt="image" src="https://github.com/albertofilice/rhacm-gitops/assets/35273403/d64106a0-8861-40d5-8894-41a8a712ff84">


**2. ApplicationSet with chart dependency**

In this case, we implemented a GIT containing a chart that exploits the Helm chart present on the Bitnami repository as a dependency.

GIT directory structure:\
```bash
example/chart-dependency/local-cluster/wordpress
    ├── Chart.yaml
    └── values.yaml
```

The Chart.yaml will contain a new chart helm with a dependency on the WordPress chart.

```yaml
apiVersion: v2
name: wordpress-custom-chart
description: A Helm chart for Kubernetes

# A chart can be either an 'application' or a 'library' chart.
#
# Application charts are a collection of templates that can be packaged into versioned archives
# to be deployed.
#
# Library charts provide useful utilities or functions for the chart developer. They're included as
# a dependency of application charts to inject those utilities and functions into the rendering
# pipeline. Library charts do not define any templates and therefore cannot be deployed.
type: application

# This is the chart version. This version number should be incremented each time you make changes
# to the chart and its templates, including the app version.
# Versions are expected to follow Semantic Versioning (https://semver.org/)
version: 0.1.0

# This is the version number of the application being deployed. This version number should be
# incremented each time you make changes to the application. Versions are not expected to
# follow Semantic Versioning. They should reflect the version the application is using.
appVersion: "1.0"

dependencies:
- name: wordpress
  version: 16.0.4
  repository: https://charts.bitnami.com/bitnami

```
The values.yaml will contain all the ovveride values, which must be contained in the name of the chart present on Bitnami and cited as a dependency.

```yaml
wordpress:
    image:
      registry: docker.io
      repository: bitnami/wordpress
      tag: 6.2.0-debian-11-r22
      digest: ""
      pullPolicy: IfNotPresent
      pullSecrets: []
      debug: false
    multisite:
      enable: false
      host: ""
      networkType: subdomain
      enableNipIoRedirect: false
```


Below is an example of ApplicationSet used for this case study:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: wordpress-chart-dependency
  namespace: openshift-gitops
spec:
  generators:
    # definition and interception of clusters via placemente RHACM.
    - clusterDecisionResource:
        configMapRef: acm-placement
        labelSelector:
          matchLabels:
            cluster.open-cluster-management.io/placement: wordpress-chart-dependency-placement
        requeueAfterSeconds: 180
  template:
    # name is autogenerated and contains the cluster name present in RHACM "name" in order to avoid duplicate applications. For example, "wordpress-chart-dependency-local-cluster."  
    metadata:
      name: wordpress-chart-dependency-{{name}}
      labels:
        velero.io/exclude-from-backup: "true"
    spec:
      # Targer of the installation, the server is a system variable and provides the API url, for example, "https://api.cluster.example.com:6443"    
      destination:
        namespace: wordpress-chart-dependency
        server: "{{server}}"
      project: default
      #Appointment to GIT containing the chart.yaml and values.yaml
      source:
        path: example/chart-dependency/{{name}}/wordpress
        repoURL: https://github.com/albertofilice/rhacm-gitops.git
        targetRevision: main
      # Defining sync options related to ArgoCD https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/     
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - PruneLast=true
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: wordpress-chart-dependency-placement
  namespace: openshift-gitops
spec:
  clusterSets:
    - sample-clusterset
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: local-cluster
              operator: In
              values:
                - "true"

```
# Final result

The end result of these implementations is the creation of an Argocd application, derived from the ApplicationSet and dynamically created based on the reference cluster configured in RedHat Advanced Cluster Manager.

<img width="1217" alt="image" src="https://github.com/albertofilice/rhacm-gitops/assets/35273403/48e00f7d-a687-42a3-b5af-c5cebe580b95">

<img width="1512" alt="image" src="https://github.com/albertofilice/rhacm-gitops/assets/35273403/748c0ed0-a5ce-433d-be25-7a19088da302">



**3. ApplicationSet with kustomize**

This solution involves implementing Kustomize for customization and Ovveride on a per-cluster basis, and in particular is applicable with the following directory structure:

```bash
example/kustomize/wordpress
├── base
│   ├── helm-chart.yaml
│   ├── kustomization.yaml
│   └── values.yaml
└── overlays
    └── local-cluster
        ├── delete-HelmChartInflationGenerator.yaml
        ├── helm-chart.yaml
        └── kustomization.yaml
```

In base will be all the common parameters applicable to each cluster, and in overlays will be the customization defined for each cluster.

In base/helm-chart.yaml is expressed the definition of the chart present in Bitnami and the reference to the value file containing the compatible values for all clusters.

```yaml
apiVersion: builtin
kind: HelmChartInflationGenerator
metadata:
  name: wordpress-kustomize
name: wordpress
repo: https://charts.bitnami.com/bitnami
version: 16.0.4
releaseName: wordpress
valuesFile: values.yaml
IncludeCRDs: false
```

values.yaml

```yaml
image:
  registry: docker.io
  repository: bitnami/wordpress
  tag: 6.2.0-debian-11-r22
  digest: ""
  pullPolicy: IfNotPresent
  pullSecrets: []
  debug: false
multisite:
  enable: false
  host: ""
  networkType: subdomain
  enableNipIoRedirect: false
...

```

base/Kustomization.yaml
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- helm-chart.yaml

generators:
- helm-chart.yaml
```
The generators feature allows you to leverage the helm plugin in Kustomize, dynamically creating a chart containing the indicated overrides. Only one overrides values.yaml can be configured. The overlays feature exploits the chart created in the base by further customizing the chart, specifically:

In overlays/helm-chart.yaml, the helm values to override per cluster are shown inline.

```yaml
apiVersion: builtin
kind: HelmChartInflationGenerator
metadata:
  name: wordpress-kustomize
  annotations:
    argocd.argoproj.io/hook: Skip  
valuesInline:
    wordpressUsername: user
    wordpressPassword: ""
    existingSecret: ""
    wordpressEmail: user@example.com
    wordpressFirstName: FirstName
    wordpressLastName: LastName
    wordpressBlogName: User's Blog!
    wordpressTablePrefix: wp_
    wordpressScheme: http

```

In the overlays/Kustomization.yaml file, the manifest is merged against the base and the defined chart is created, which will then be installed in the clusters.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

patchesStrategicMerge:
  - helm-chart.yaml
  - delete-HelmChartInflationGenerator.yaml

resources:
- ../../base
```
The file overlays/delete-HelmChartInflationGenerator.yaml is useful in order to avoid a loop in the merge of the HelmChartInflationGenerator resource. Therefore, this resource is deleted and not configured in Argocd, as this resource is only exploited by the Kustomize build for chart creation.
```yaml
$patch: delete
apiVersion: builtin
kind: HelmChartInflationGenerator
metadata:
  name: wordpress-kustomize
```

Below is an example of ApplicationSet used for this case study:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: wordpress-kustomize
  namespace: openshift-gitops
spec:
  generators:
    # definition and interception of clusters via placemente RHACM.
    - clusterDecisionResource:
        configMapRef: acm-placement
        labelSelector:
          matchLabels:
            cluster.open-cluster-management.io/placement: wordpress-kustomize-placement
        requeueAfterSeconds: 180
  template:
    # name is autogenerated and contains the cluster name present in RHACM "{{name}}" in order to avoid duplicate applications. For example, "wordpress-kustomize-local-cluster"
    metadata:
      name: wordpress-kustomize-{{name}}
      labels:
        velero.io/exclude-from-backup: "true"
    spec:
      # Targer of the installation, the server is a system variable and provides the API url, for example, "https://api.cluster.example.com:6443" 
      destination:
        namespace: wordpress-kustomize
        server: "{{server}}"
      project: default     
      # Pointing to git, specifically to the specific overlays path. 
      source:   
        path: example/kustomize/wordpress/overlays/{{name}}
        repoURL: https://github.com/albertofilice/rhacm-gitops.git
        targetRevision: main
      # Defining sync options related to ArgoCD: https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - PruneLast=true
---
apiVersion: cluster.open-cluster-management.io/v1beta1
kind: Placement
metadata:
  name: wordpress-kustomize-placement
  namespace: openshift-gitops
spec:
  clusterSets:
    - sample-clusterset
  predicates:
    - requiredClusterSelector:
        labelSelector:
          matchExpressions:
            - key: local-cluster
              operator: In
              values:
                - "true"

```
# Final result

The end result of these implementations is the creation of an Argocd application, derived from the ApplicationSet and dynamically created based on the reference cluster configured in RedHat Advanced Cluster Manager.

<img width="1512" alt="image" src="https://github.com/albertofilice/rhacm-gitops/assets/35273403/1eaa9bf7-c5f8-4b4f-aff9-b5105dc446b9">

<img width="1512" alt="image" src="https://github.com/albertofilice/rhacm-gitops/assets/35273403/8f109c7c-daaf-422f-9137-cd15bd1fa873">
