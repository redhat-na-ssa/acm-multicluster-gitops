# Clusters as Code with ACM and GitOps

This repo demonstrates managing multiple OpenShift clusters with ACM and GitOps.
Useful for demonstrating cluster-level multitenancy.

## Directory structure

```
├── bootstrap
│   ├── clusters
│   └── operators
├── clusters
│   ├── team-1
│   ├── team-2
│   └── team-3
└── components
    ├── applicationset
    ├── managedcluster
    │   └── aws
    ├── operators
    │   ├── cert-manager
    │   └── web-terminal
    └── secrets
        ├── cloud_credentials
        └── installconfigs
            └── aws
```

- `bootstrap`: Components that initialize the ACM hub to manage things with
  GitOps.
  - `bootstrap/clusters`: Managed clusters to provision and the ArgoCD
    ApplicationSets that will configure them.
  - `bootstrap/operators`: Operators to install on the ACM hub.
- `clusters`: A list of clusters being managed by GitOps.
- `components`: Re-usable components that can be used by the ACM hub and any
  managed clusters it manages.
  - `components/applicationset`: Creates an ArgoCD Application Set.
  - `managedcluster/$CLOUD`: Provisions a managed cluster in ACM.
  - `operators`: A list of operators that can be managed with GitOps.
  - `secrets`: Secrets to install into OpenShift clusters.

> **NOTE**: Secrets in the `secrets` directory are NOT commited to Git. See the
> `examples` directory for examples.

## How to use this demo

### Prerequisites

- An OpenShift cluster with ACM and Multicluster Engine installed.
- `oc` or `kubectl`

### Deploying

Copy the example secrets into the `secrets` top-level directory:

```sh
find components/secrets/examples -type f |
while read -r secret
do
  target="${secrets//example\//}"
  mkdir -p "$(basename "$target")"
  cp "$secret" "$target"
done
```

Render the Kustomize templates in the `bootstrap` directory:

```sh
oc kustomize bootstrap
```

This should produce a YAML like the one shown below:

```yaml
```

If it does, apply it and watch it go!

```sh
oc apply -k bootstrap
```

Wait about an hour for the three managed clusters to finish provisioning.

You should see something like the below:

`#WIP`
