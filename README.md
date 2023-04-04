# What is this repo for

Each Kubernetes cluster requires a basic set of services installed to make it useful. These are things
such as logging, monitoring, ingress etc. Rather than manually installing these services, or using
bash scripts, this repo is meant for a tool called ArgoCD to automatically apply and keep in sync
all these resources.

This lets us simply push new changes, or even new services to this repository and they'll be
automatically applied to the clusters.

ArgoCD uses a pull based system so it pulls down the changes itself rather than a CI job pushing
the changes.

# Whats in this application set

* Cert-manager
* ElasticSearch logging
* Prometheus monitoring
* Persistent Pod Storage

# Installing ArgoCD into new clusters

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Change the ApplicationSet policy to not delete Applications automatically.
# This helps prevent errors from deleting all our infrastructure.
kubectl patch configmap/argocd-cmd-params-cm \
        -n argocd \
        --type merge \
        -p '{"data":{"applicationsetcontroller.policy":"create-update"}}'
kubectl rollout restart deployment argocd-applicationset-controller -n argocd

# Applies the list of Applications we want to install
```

For easier external access it's possible to do:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Or simply

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

and access via `https://localhost:8080`.

Extra instructions are on the ArgoCD site - https://argo-cd.readthedocs.io/en/stable/getting_started/

To get the admin password you can run:

```
argocd admin initial-password -n argocd
```

The local command line tool needs installing for this to work which is described in the above docs. Or you can grab it directly from
the `argocd-secret` Secret resource from the `admin.password` key (remember secrets are BASE64 encoded so you need to pipe the value
into `base64 -d`).

# ArgoCD Setup (for hosted Gitlab)

Add the following known host for SSH Git access to Settings -> Repository Certificates and Known Hosts:

```
gitlab.domain ecdsa-sha2-nistp256 AAAA...=
```

Add the following SSH key info to Settings -> Repository -> Connect Repo:

* Connection Method: VIA SSH
* Name: Choose any name
* Project: Default
* Repository URL: git@gitlab.domain:repo.git
* SSH Private Key: see below

Click Connect. The Connection Status column should say successful.

To create a private key you need to add a deploy key to your repo containing your application data yaml (this repo).

Gitlab docs on this are at https://docs.gitlab.com/ee/user/project/deploy_keys/. The key doesn't need write permission. I used this
command to create a key `ssh-keygen -t ed25519`.

Once you've created your key, paste the public key into Gitlab as instructed above, and the private key should be pasted into the
ArgoCD Settings above.

# Bootstrapping new clusters

```bash
kubectl apply -f applications-dev.yaml -n argocd`
```

This installs all the apps defined in this repo into the cluster.

Each cluster has a unique file to apply to it. *IMPORTANT* only one `ApplicationSet` should be deployed per cluster.

# Directory Layout & Naming

```
.
├── applications-dev.yaml
├── applications-prod.yaml
├── charts
│   ├── cert-manager
│   │   └── cert-manager
│   │       ├── dev
│   │       │   ├── Chart.yaml
│   │       │   ├── templates
│   │       │   │   ├── sertigo-clusterissuer.yaml
│   │       │   │   └── sertigo-secret.yaml
│   │       │   └── values.yaml
│   │       ├── prod
│   │       │   ├── Chart.yaml
│   │       │   ├── templates
│   │       │   │   ├── sertigo-clusterissuer.yaml
│   │       │   │   └── sertigo-secret.yaml
│   │       │   └── values.yaml
│   │       └── values-common.yaml
└── resources
    └── storage
        └── storageclass
            ├── dev
            │   └── tkgi-storageclass.yaml
            └── prod
                └── tkgi-storageclass.yaml
```

At the top is the `ApplicationSet` for each different cluster. Each cluster needs its own file so that it can set the environment name to install charts or resources from.

Any helm charts go in the `charts` directory, and any raw yaml files go in the `resources` directory.

The next level down is the name of the `namespace` to deploy in. This will be auto-created. Underneath is the name of the release. This lets
us put multiple releases in the same namespace if required. Additionally, in this directory a `values.common.yaml` file lets us share helm config values between environments.

Next level is the environment/cluster we're deploying to. This is chosen based on which of the `ApplicationSet`s is actually deployed in the cluster.

Now inside here goes a `Chart.yaml` (or the raw yaml files if under `resources` instead). Any additional templates to add go in a `templates` directory here too.

Any value overrides for this particular environment go in `values.yaml` here.

*IMPORTANT* for the values file, this works exactly the same way as a helm chart which has dependencies. What that means is any key/value that would normally go in the `values.yaml` (or the `values-common.yaml`) needs the top level key to be the name of the chart. Eg for `cert-manager`, it has a key `prometheus.servicemonitor.enabled`. In the values file it should look like:

```yaml
cert-manager:
  prometheus:
    servicemonitor:
      enabled: true
```

The `cert-manager` key corresponds to the `name` in the `Chart.yaml`.

This hierarchy is defined in `applications-env.yaml`. `{{path[1]}}` is the namespace and `{{path[2]}}` is the name of the deployment. `{{path[0]}}` would be `charts` in this case.

*IMPORTANT* namespaces aren't deleted if you delete the app. This needs doing manually, although if you do this then you lose your IP that the pods access the rest of the network as (see below).

# Backups/Restoring

See https://argo-cd.readthedocs.io/en/stable/operator-manual/disaster_recovery/

tldr; `argocd admin export > backup.yaml` and `argocd admin import - < backup.yaml`

This doesn't backup the argo application itself, just config plus any applications that are deployed.
