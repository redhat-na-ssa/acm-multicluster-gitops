# Clusters as Code with ACM and GitOps

This repo demonstrates managing multiple OpenShift clusters with ACM and GitOps.
Useful for demonstrating cluster-level multitenancy.

## Directory structure

```
├── bootstrap
│   ├── base
│   ├── namespaces
│   │   └── clusters
│   │       ├── team-1
│   │       ├── team-2
│   │       └── team-3
│   └── secrets
│       └── clusters
│           ├── team-1
│           ├── team-2
│           └── team-3
├── cluster_typess
│   ├── type_a
│   ├── type_b
│   └── type_c
├── components
│   ├── application
│   ├── applicationset
│   ├── managedcluster
│   │   └── aws
│   ├── namespace
│   ├── operators
│   │   ├── _base
│   │   ├── cert-manager
│   │   └── web-terminal
│   └── secrets
│       ├── _examples
│       │   ├── cloud_credentials
│       │   │   └── aws
│       │   ├── installconfig
│       │   │   └── aws
│       │   └── pull_secret
│       ├── cloud_credentials # <-- NOT COMMITTED TO GIT!
│       │   └── aws
│       ├── installconfig # <-- NOT COMMITTED TO GIT!
│       │   ├── aws
│       │   └── templates
│       └── pull_secret # <-- NOT COMMITTED TO GIT!
└── hubs
    └── primary
        ├── managed_clusters
        │   └── team-1
        │       ├── gitops
        │       ├── managedcluster
        │       └── secrets
        └── operators

```

- `bootstrap`: Resources used to kickstart GitOps on ACM hubs.
    - `bootstrap/namespace`: Namespaces that are applied onto ACM hubs **manually**
      before applying GitOps resources.
    - `bootstrap/secrets`: Secrets that are applied onto ACM hubs **manually**
      before applying GitOps resources.
    - `bootstrap/base`: The resource that kickstarts GitOps.
- `cluster_types`: A portfolio of different "flavors" of clusters that your
  platform teams support. This is useful for ensuring consistent cluster
  configuration for multiple teams.
- `components`: Resources deployed into ACM hubs or managed clusters, like
  namespaces, ArgoCD configurations, operators, and so on.
- `hubs`: ACM hubs. The bootstrap process creates an ArgoCD application that
  synchronizes these subdirectories with hub clusters.
  - `hubs/$HUB_NAME/managed_clusters`: Clusters managed by an ACM hub.
  - `hubs/$HUB_NAME/operators`: Operators installed into the hub.


## How to use this demo

### Prerequisites

- An OpenShift cluster with ACM and Multicluster Engine installed.
- `oc` or `kubectl`
- A GitHub or GitLab Account (any Git repo will work as long as your ACM hub can
  access it)

### Generating Secrets

> **NOTE**: While we are generating the secrets "by hand" here, in a real-world
> scenario, you would use a secrets manager like HashiCorp Vault to store these
> secrets securely elsewhere or an encryption tool like
> [sops](https://github.com/getsops/sops) to store them securely in a Git
> repository.

#### Creating the directory structure

Copy the example secrets into the `secrets` top-level directory:

```sh
find components/secrets/_examples -type f |
while read -r secret
do
  target="${secret//_examples\//}"
  mkdir -p "$(dirname "$target")"
  cp "$secret" "$target"
done
```

> **NOTE**: These are **NOT** committed to Git!

#### Updating the Installer Config secret for your managed clusters

Next, copy the example Installer config in `components/secrets/installconfigs/templates/`
to your `/tmp` directory, then open it in an editor and replace the keys set to
`replace-me` with real values.

Finally, use the command below to update the Secret that will hold your
Installer Config with a base64 representation:

```sh
yq -ir '.data."install-config.yaml" = "'$(base64 -w=0 < /tmp/install-config.yaml)'"' \
  components/secrets/installconfig/aws/installconfig.yaml
```

#### Updating cloud credential secrets

Open `components/secrets/cloud_credentials/aws/credential.yaml` and replace the
keys set to `replace-me` with real values.

#### Updating the OCP pull secret secret

1. Retrieve the pull secret for the Red Hat registry or your company's private
   registry. It should look something like the below:

    ```json
    {"auths":{"cloud.openshift.com":{"email":"example@email.address","auth":"long-string"}}}
    ```

2. Run the command below to update the pull secret credential at
   `components/secrets/pull_secret/credential.yaml` with a base64-encoded
   representation:

   ```sh
   yq -r '.data.".dockerconfigjson" = "'$(echo "$YOUR_PULL_SECRET"| base64 -w=0)'"' ]
    components/secrets/pull_secret/credential.yaml
   ```

### Validating bootstrap secrets

Render the Kustomize template that will apply secrets to your cluster:

```sh
oc kustomize bootstrap/secrets
```

You should see a YAML like shown below:

```yaml
apiVersion: v1
data:
  aws_access_key_id: some key
  aws_secret_access_key: some value
kind: Secret
metadata:
  labels:
    cluster.open-cluster-management.io/backup: ""
    cluster.open-cluster-management.io/credentials: ""
    cluster.open-cluster-management.io/type: aws
  name: cloud-creds
  namespace: cluster-team-1
---
apiVersion: v1
data:
  install-config.yaml: some-install-config
kind: Secret
metadata:
  labels:

```

Verify that `replace-me` is not in the output:

```sh
oc kustomize bootstrap/secrets | grep -i replace
# should be empty
```

### Validating Hub configuration

Render the Kustomize template that ArgoCD will apply onto your primary ACM hub:

```sh
oc kustomize hubs/primary
```

This should render a YAML similar to this:

```yaml
apiVersion: operators.coreos.com/v1alpha1
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-operator
---
apiVersion: v1
kind: Namespace
metadata:
  name: cluster-team-1
---
apiVersion: v1
kind: Namespace
metadata:
  name: web-terminal-operator
---
apiVersion: agent.open-cluster-management.io/v1
kind: KlusterletAddonConfig
metadata:
  name: cluster-team-1
  namespace: cluster-team-1
spec:
  applicationManager:
    enabled: true
# ...omitted for brevity
```

Verify that `replace-me` is not in the output:

```sh
$: kubectl kustomize hubs/primary | grep replace-me
# should be empty
```

### Validating the bootstrap configuration

Render the Kustomize templates in the `bootstrap` directory:

```sh
oc kustomize bootstrap/base
```

This should produce a YAML like the one shown below:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: primary-hub
  namespace: openshift-gitops
spec:
  destination:
    namespace: primary-hub-bootstrap
    server: https://kubernetes.default.svc
  ignoreDifferences:
  - group: cluster.open-cluster-management.io
    jsonPointers:
    - /metadata/labels/cloud
    - /metadata/labels/name
    - /metadata/labels/vendor
    kind: ManagedCluster
  - group: operator.open-cluster-management.io
    jsonPointers:
    - /spec/overrides/components
    kind: MultiClusterHub
  - group: multicluster.openshift.io
    jsonPointers:
    - /spec/overrides/components
    kind: MultiClusterEngine
  project: default
  source:
    path: ./hubs/primary
    repoURL: https://github.com/redhat-na-ssa/acm-gitops-demo
    targetRevision: main
  syncPolicy:
    automated: {}
    syncOptions:
    - CreateNamespace=true
    - SkipDryRunOnMissingResource=true
    - RespectIgnoreDifferences=true
```

### Deploy!

If everything checks out, bootstrap your ACM hub and watch it go!

```sh
oc apply -k bootstrap/base
```

Wait about an hour for the three managed clusters to finish provisioning.

You should see something like the below:

`#WIP`

## Customizations

### Using a private repository

Do the following if you'd like to synchronize your cluster with a private
repository:

1. Update the `sshPrivateKey` property in
   `components/secrets/argocd_repository/credentials.yaml` with the SSH
   **private** key for your repository.
2. Replace the `replace-me` values in
   `bootstrap/secrets/argocd-private-repository/kustomization.yaml` with actual
   values.
3. Add the below under `resources` in the
   `bootstrap/secrets/kustomization.yaml`:

   ```sh
   - argocd-private-repository
   ```
4. Replace all references to repositories in this codebase with the URL to your
   private repo:

   ```sh
   url="your-repo-url" # <--------- replace this
   grep -lr repoURL |
    grep -v README.md |
    xargs sed -Ei "s;repoURL:.*;repoURL: $url;g"
   ```

## Extending the demo

GitOps enables you to customize any of your clusters to your liking! Here are
some examples of tasks you can perform after you begin managing your clusters
with GitOps.

### Create a "staging" ACM hub

You might want to create an ACM hub for testing new clusters, policies and/or
configurations. This is straightforward to do with GitOps.

1. Create a new copy of `hubs/primary`: `cp hubs/primary hubs/staging`
2. Create a new bootstrap base for your new hub: `cp bootstrap/base
   bootstrap/staging`
3. Remove (or add!) any managed clusters in `hubs/staging`.
4. Commit and push your changes.
5. Bootstrap the new ACM hub: `oc login api.$STAGING_CLUSTER && oc apply -k
   bootstrap/staging`

### Provisioning clusters for teams

1. Create a new branch off of `main`.
2. Create a new copy of `hubs/primary/managed_clusters/team-1`.
3. Modify the patches in the managed cluster kustomization that you copied.
4. Commit and push your changes into your branch.
5. Create a pull request to merge your branch into `main`.
6. Hold a review session to verify your changes with the team (or other
   stakeholders).
7. Approve the request. The new cluster will get created within a minute or two
   of `main` being updated.

> **NOTE**: This approach works well with a "staging" ACM hub, especially if
> teams will be requesting new clusters in a self-service model.
