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
            └── aws
        └── installconfigs
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

### Generating Secrets

> **NOTE**: While we are generating the secrets "by hand" here, in a real-world
> scenario, you would use a secrets manager like HashiCorp Vault to store these
> secrets securely elsewhere or an encryption tool like
> [sops](https://github.com/getsops/sops) to store them securely in a Git
> repository.

#### Creating the directory structure

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

#### Updating the Installer Config secret for your managed clusters

Next, copy the example Installer config in `components/secrets/installconfigs/templates/`
to your `/tmp` directory, then open it in an editor and replace the keys set to
`replace-me` with real values.

Finally, use the command below to update the Secret that will hold your
Installer Config with a base64 representation:

```sh
yq -r '.data."install-config.yaml" = "'$(base64 -w=0 < /tmp/install-config.yaml)'"' \
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

### Bootstrapping GitOps

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
