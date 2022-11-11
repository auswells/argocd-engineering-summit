# Example ArgoCD GitOps Repository

## ArgoCD Setup

Create argocd namespace
```shell
kubectl apply -f environments/aw-es-argocd/argocd/namespace.yaml
```

Install argocd
```shell
kubectl apply -f environments/aw-es-argocd/argocd/install.yaml
```

Example secret to register a cluster with argocd  
Ref: https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#clusters  
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mycluster-secret
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: mycluster.com
  server: https://mycluster.com
  config: |
    {
      "bearerToken": "<authentication token>",
      "tlsClientConfig": {
        "insecure": false,
        "caData": "<base64 encoded certificate>"
      }
    }
```

Example secret to connect a private repository to argocd  
Ref: https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#repositories  
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: private-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/argoproj/private-repo
  password: my-password
  username: my-username
```

Example application manifest  
Ref: https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#applications  
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  # You'll usually want to add your resources to the argocd namespace.
  namespace: argocd
  # Add this finalizer ONLY if you want these to cascade delete.
  finalizers:
    - resources-finalizer.argocd.argoproj.io
  # Add labels to your application object.
  labels:
    name: guestbook
spec:
  # The project the application belongs to.
  project: default

  # Source of the application manifests
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git  # Can point to either a Helm chart repo or a git repo.
    targetRevision: HEAD  # For Helm, this refers to the chart version.
    path: guestbook  # This has no meaning for Helm charts pulled directly from a Helm repo instead of git.

    # helm specific config
    chart: chart-name  # Set this when pulling directly from a Helm repo. DO NOT set for git-hosted Helm charts.
    helm:
      passCredentials: false # If true then adds --pass-credentials to Helm commands to pass credentials to all domains
      # Extra parameters to set (same as setting through values.yaml, but these take precedence)
      parameters:
        - name: "nginx-ingress.controller.service.annotations.external-dns\\.alpha\\.kubernetes\\.io/hostname"
          value: mydomain.example.com
        - name: "ingress.annotations.kubernetes\\.io/tls-acme"
          value: "true"
          forceString: true # ensures that value is treated as a string

      # Use the contents of files as parameters (uses Helm's --set-file)
      fileParameters:
        - name: config
          path: files/config.json

      # Release name override (defaults to application name)
      releaseName: guestbook

      # Helm values files for overriding values in the helm chart
      # The path is relative to the spec.source.path directory defined above
      valueFiles:
        - values-prod.yaml

      # Values file as block file
      values: |
        ingress:
          enabled: true
          path: /
          hosts:
            - mydomain.example.com
          annotations:
            kubernetes.io/ingress.class: nginx
            kubernetes.io/tls-acme: "true"
          labels: {}
          tls:
            - secretName: mydomain-tls
              hosts:
                - mydomain.example.com

      # Optional Helm version to template with. If omitted it will fall back to look at the 'apiVersion' in Chart.yaml
      # and decide which Helm binary to use automatically. This field can be either 'v2' or 'v3'.
      version: v2

    # kustomize specific config
    kustomize:
      # Optional kustomize version. Note: version must be configured in argocd-cm ConfigMap
      version: v3.5.4
      # Supported kustomize transformers. https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/
      namePrefix: prod-
      nameSuffix: -some-suffix
      commonLabels:
        foo: bar
      commonAnnotations:
        beep: boop
      images:
        - gcr.io/heptio-images/ks-guestbook-demo:0.2
        - my-app=gcr.io/my-repo/my-app:0.1

    # directory
    directory:
      recurse: true
      jsonnet:
        # A list of Jsonnet External Variables
        extVars:
          - name: foo
            value: bar
            # You can use "code to determine if the value is either string (false, the default) or Jsonnet code (if code is true).
          - code: true
            name: baz
            value: "true"
        # A list of Jsonnet Top-level Arguments
        tlas:
          - code: false
            name: foo
            value: bar
      # Exclude contains a glob pattern to match paths against that should be explicitly excluded from being used during
      # manifest generation. This takes precedence over the `include` field.
      # To match multiple patterns, wrap the patterns in {} and separate them with commas. For example: '{config.yaml,env-use2/*}'
      exclude: 'config.yaml'
      # Include contains a glob pattern to match paths against that should be explicitly included during manifest
      # generation. If this field is set, only matching manifests will be included.
      # To match multiple patterns, wrap the patterns in {} and separate them with commas. For example: '{*.yml,*.yaml}'
      include: '*.yaml'

    # plugin specific config
    plugin:
      # Only set the plugin name if the plugin is defined in argocd-cm.
      # If the plugin is defined as a sidecar, omit the name. The plugin will be automatically matched with the
      # Application according to the plugin's discovery rules.
      name: mypluginname
      # environment variables passed to the plugin
      env:
        - name: FOO
          value: bar

  # Destination cluster and namespace to deploy the application
  destination:
    server: https://kubernetes.default.svc
    # The namespace will only be set for namespace-scoped resources that have not set a value for .metadata.namespace
    namespace: guestbook

  # Sync policy
  syncPolicy:
    automated: # automated sync by default retries failed attempts 5 times with following delays between attempts ( 5s, 10s, 20s, 40s, 80s ); retry controlled using `retry` field.
      prune: true # Specifies if resources should be pruned during auto-syncing ( false by default ).
      selfHeal: true # Specifies if partial app sync should be executed when resources are changed only in target Kubernetes cluster and no git change detected ( false by default ).
      allowEmpty: false # Allows deleting all application resources during automatic syncing ( false by default ).
    syncOptions:     # Sync options which modifies sync behavior
      - Validate=false # disables resource validation (equivalent to 'kubectl apply --validate=false') ( true by default ).
      - CreateNamespace=true # Namespace Auto-Creation ensures that namespace specified as the application destination exists in the destination cluster.
      - PrunePropagationPolicy=foreground # Supported policies are background, foreground and orphan.
      - PruneLast=true # Allow the ability for resource pruning to happen as a final, implicit wave of a sync operation
    # The retry feature is available since v1.7
    retry:
      limit: 5 # number of failed sync attempt retries; unlimited number of attempts if less than 0
      backoff:
        duration: 5s # the amount to back off. Default unit is seconds, but could also be a duration (e.g. "2m", "1h")
        factor: 2 # a factor to multiply the base duration after each failed retry
        maxDuration: 3m # the maximum amount of time allowed for the backoff strategy

  # Will ignore differences between live and desired states during the diff. Note that these configurations are not
  # used during the sync process.
  ignoreDifferences:
    # for the specified json pointers
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas
    # for the specified managedFields managers
    - group: "*"
      kind: "*"
      managedFieldsManagers:
        - kube-controller-manager

  # RevisionHistoryLimit limits the number of items kept in the application's revision history, which is used for
  # informational purposes as well as for rollbacks to previous versions. This should only be changed in exceptional
  # circumstances. Setting to zero will store no history. This will reduce storage used. Increasing will increase the
  # space used to store the history, so we do not recommend increasing it.
  revisionHistoryLimit: 10

```
