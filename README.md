# k8s-fluxcd-wordpress

This is for an assignment where we achieve the below:

- Create a clean Kubernetes cluster (minikube).
- Use FluxCD for GitOps and Helm for deploying WordPress.
- Automate the process of image updates for WordPress using FluxCD's automated sync mechanism.
- Write documentation to explain your setup and deployment process.

We will also harden the solution further and have multi environment support which the repository structure is based on.


## Prerequisites

You will need a container or virtual machine manager such as Docker.

For the purpose of this assignment we will go with Docker - https://www.docker.com/products/docker-desktop/

You will need a Kubernetes cluster version 1.26 or newer.
We will be using minikube.
Any other Kubernetes setup will work as well though.

```sh
brew install minikube
```

In order to follow the guide you'll need a GitHub account and a
[personal access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line)
that can create repositories (check all permissions under `repo`).

Install the Flux CLI on macOS or Linux using Homebrew:

```sh
brew install fluxcd/tap/flux
```

Or install the CLI by downloading precompiled binaries using a Bash script:

```sh
curl -s https://fluxcd.io/install.sh | sudo bash
```

## Repository structure

The Git repository contains the following top directories:

- **apps** dir contains Helm releases with a custom configuration per cluster
- **clusters** dir contains the Flux configuration per cluster

```
├── apps
│   ├── base
│   ├── dev 
│   └── prod
│
└── clusters
    ├── dev
    └── prod
```

### Applications

The apps configuration is structured into:

- **apps/base/** dir contains namespaces and Helm release definitions
- **apps/dev/** dir contains the dev environment Helm release values
- **apps/prod/** dir contains the prod environment Helm release values

```
./apps/
├── base
│   └── wordpress
│       ├── kustomization.yaml
│       ├── release.yaml
│       └── repository.yaml
├── dev
│   ├── kustomization.yaml
│   └── wordpress-values.yaml
└── prod
    ├── kustomization.yaml
    └── wordpress-values.yaml
```

In **apps/base/wordpress/** dir we have a Flux `HelmRelease` with common values for both clusters:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: wordpress
  namespace: default
spec:
  interval: 5m
  chart:
    spec:
      chart: wordpress
      version: "20.1.2"
      sourceRef:
        kind: HelmRepository
        name: bitnami
        namespace: default
      interval: 1m
  values:
    image:
      repository: bitnami/wordpress
      tag: 6.4.3 # {"$imagepolicy": "flux-system:wordpress-policy:tag"}
    mariadb:
      auth:
        rootPassword: "adminPass123"
        database: "wordpress"
    wordpressUsername: "admin"
    wordpressPassword: "password123"
    wordpressEmail: "user@example.com"
    service:
      type: ClusterIP
```

In **apps/dev/** dir we have a Kustomize patch with the development environment specific values:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: wordpress
spec:
  values:
    replicaCount: 2
    mariadb:
      primary:
        persistence:
          size: "5Gi"
    service:
      type: ClusterIP
```

Note that with ` replicaCount: 2` we configure Flux to automatically upgrade
the `HelmRelease` to deploy 2 replicas for this environment. We are also setting a 5GB database size with ` size: "5Gi`.

In **apps/prod/** dir we have a Kustomize patch with the production specific values:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: wordpress
spec:
  values:
    replicaCount: 5
    mariadb:
      primary:
        persistence:
          size: "10Gi"
    service:
      type: ClusterIP
```

Note that with ` replicaCount: 5` we configure Flux to automatically upgrade
the `HelmRelease` to deploy 5 replicas for this environment. We are also setting a 10GB database size with ` size: "10Gi`.


## Bootstrap dev and prod

The clusters dir contains the Flux configuration:

```
./clusters/
├── dev
│   ├── apps.yaml
│   └── wordpress-imageautomation.yaml
│   └── wordpress-imagepolicy.yaml
│   └── wordpress-imagerepo.yaml
│
└── prod
    ├── apps.yaml
    └── wordpress-imageautomation.yaml
    └── wordpress-imagepolicy.yaml
    └── wordpress-imagerepo.yaml
```

In **clusters/dev/** dir we have the Flux Kustomization definitions, for example the apps.yaml:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/dev
  prune: true
  wait: true
  timeout: 5m0s
```

Note that with `path: ./apps/dev` we configure Flux to sync the dev environment Kustomize overlay.
We also have 3 wordpress related image automation files which we will talk about later.

Fork this repository on your personal GitHub account and export your GitHub access token, username and repo name:

```sh
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
export GITHUB_REPO=<repository-name>
```

Verify that your dev cluster satisfies the prerequisites with:

```sh
flux check --pre
```

Set the kubectl context to your dev cluster and bootstrap Flux:

```sh
  flux bootstrap github \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --token-auth \
    --path=./clusters/dev \
    --components-extra=image-reflector-controller,image-automation-controller
```

The bootstrap command commits the manifests for the Flux components in `clusters/dev/flux-system` dir
and creates a deploy key with read-only access on GitHub, so it can pull changes inside the cluster.

Watch for the Helm releases being installed on dev:

```console
$ watch flux get helmreleases --all-namespaces

NAMESPACE    	NAME         	REVISION	SUSPENDED	READY	MESSAGE 
default      	wordpress    	20.1.2   	False    	True 	Helm install succeeded for release default/wordpress.v1 with chart wordpress@20.1.2
```

Verify that the demo wordpress app can be accessed:

```console
$ kubectl get pods --namespace default

$ kubectl port-forward pod/wordpress-67c6c5bd76-r7dg6 8080:8080 --namespace default
```

After the above you should be able to hit the pod from browser using - http://localhost:8080


You can Bootstrap Flux on prod by setting the context and path to your production cluster:

```sh
  flux bootstrap github \
    --owner=${GITHUB_USER} \
    --repository=${GITHUB_REPO} \
    --branch=main \
    --personal \
    --token-auth \
    --path=./clusters/prod \
    --components-extra=image-reflector-controller,image-automation-controller
```


## Image Automation
For a detailed guide please visit - https://fluxcd.io/flux/guides/image-update/

We have 3 image automation yaml files for each environment. Lets look at the ones for the dev environment.

In **clusters/dev/** dir we have the wordpress-imagerepo.yaml:

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: wordpress-repo
  namespace: flux-system
spec:
  image: docker.io/bitnami/wordpress
  interval: 1h
```

This will tell Flux which container registry to scan for new image tags.

Next in **clusters/dev/** dir we have the wordpress-imagepolicy.yaml:

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: wordpress-policy
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: wordpress-repo
  policy:
    semver:
      range: ">=6.0.0"
```

Above we have a policy instructing Flux to apply the latest image tag and the minimum version has to always be 6.x.x

Finally in **clusters/dev/** dir we have the wordpress-imageautomation.yaml:

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta1
kind: ImageUpdateAutomation
metadata:
  name: wordpress-auto
  namespace: flux-system
spec:
  interval: 1h
  sourceRef:
    kind: GitRepository
    name: flux-system
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        name: fluxcdbot
        email: fluxcdbot@example.com
      messageTemplate: "update WordPress image to {{range .Updated.Images}}{{println .}}{{end}}"
  update:
    path: ./apps/base/wordpress
    strategy: Setters
```

The image automation yaml above will run all the policies at the specified interval. In our case it will check every hour for newer image versions and apply the change if necessary. It will also commit this change back to git within the ` apps/base/wordpress/release.yaml` file.

Specifically on the tag line:

```yaml
  values:
    image:
      repository: bitnami/wordpress
      tag: 6.4.3 # {"$imagepolicy": "flux-system:wordpress-policy:tag"}
```

For example if we had ` tag: 6.1.1` there initially, then the image automation would have automatically scanned the registry for new images, downloaded the latest one, committed the new tag version to the release.yaml file and then flux would have picked up this change and updated the image within the cluster putting us on the latest version as needed.

## References
- https://www.docker.com/products/docker-desktop/
- https://minikube.sigs.k8s.io/docs/start/
- https://fluxcd.io/flux/installation/
- https://fluxcd.io/flux/installation/bootstrap/github/
- https://fluxcd.io/flux/guides/image-update/
- https://fluxcd.io/flux/guides/repository-structure/





