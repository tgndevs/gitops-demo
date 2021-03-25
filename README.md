# gitops-demo

[![Demo App CI](https://github.com/tgndevs/gitops-demo/actions/workflows/demoapp-ci.yml/badge.svg)](https://github.com/tgndevs/gitops-demo/actions/workflows/demoapp-ci.yml)
[![Update Flux](https://github.com/tgndevs/gitops-demo/actions/workflows/update-flux.yml/badge.svg)](https://github.com/tgndevs/gitops-demo/actions/workflows/update-flux.yml)

This repository contains the artifacts used in demo part of the [Tarragona Developers Meetup event from 25/03/2021](https://www.meetup.com/Tarragona-Developers-Meetup/events/276963798/).

The repository is structured as follows.

- `.github/workflows` contains the `ci.yml` GitHub Action to build and push the Docker image for the demo application and the `update-flux.yml` Action that runs periodically and checks for new Flux releases and updates the Flux manifests in `clusters/GitopsAks/flux-system`.
- `app` contains the demo app (borrowed from [paulbouwer/hello-kubernetes](https://github.com/paulbouwer/hello-kubernetes)).
- `clusters/GitopsAks` contains the Flux system components and Flux configuration for the demo app (e.g. git source and Docker registry connectivity, alerts, image update policy, etc.).
- `manifests` contains the manifests needed to run the demo app.

## Create a Kubernetes cluster

For the demo we used a cluster in [Azure Kubernetes Service (AKS)](https://azure.microsoft.com/en-us/services/kubernetes-service/), but you can use any other Kubernetes service provider or spin up a local cluster with [kind](https://kind.sigs.k8s.io).

After you have set up your cluster and connected to it follow the steps below to bootstrap the cluster with Flux and run a demo application.

## Set up Flux

Install Flux CD locally.

```shell
curl -s https://toolkit.fluxcd.io/install.sh | sudo bash
```

Validate Flux pre-requisites.

```shell
flux check --pre
```

Export your GitHub username and Personal Access Token with full access to the repo scope and the cluster name.

```shell
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
export CLUSTER_NAME=GitopsAks
```

Bootstrap the Flux system components.

```shell
flux bootstrap github \
  --components-extra=image-reflector-controller,image-automation-controller \
  --owner=tgndevs \
  --repository=gitops-demo \
  --branch=main \
  --path=./clusters/$CLUSTER_NAME
```

Check that Flux components have been installed correctly.

```shell
flux check
```

## Set up the application

Create the app namespace.

```
kubectl create namespace tgndevs
```

Create the secret containing the basic authentication for the app sources. In this case we are using the same repository for the Flux system and the app, but they could be different.

```
flux create secret git demoapp \
  --namespace=tgndevs \
  --url=https://github.com/tgndevs/gitops-demo \
  --username=$GITHUB_USER \
  --password $GITHUB_TOKEN
```

Create the app source. This will create a `GitRepository` resource manifest.

```
flux create source git demoapp \
  --namespace=tgndevs \
  --url=https://github.com/tgndevs/gitops-demo \
  --branch=main \
  --interval=30s \
  --secret-ref=demoapp \
  --export > ./clusters/$CLUSTER_NAME/demoapp-source.yaml
```

Create a Kustomization to sync your app manifests.

```
flux create kustomization demoapp \
  --namespace=tgndevs \
  --source=demoapp \
  --path="./manifests" \
  --prune=true \
  --validation=client \
  --interval=5m \
  --export > ./clusters/$CLUSTER_NAME/demoapp-kustomization.yaml
```

After a few moments you should see how your Kustomization has been applied and it's ready and the app should start to be deployed.

```
$ kubectl -n tgndevs get kustomizations
NAME       READY   STATUS                                                            AGE
demoapp    True    Applied revision: main/dc39cc3383e8d2e8cee71658fccdddbd72fb7438   2m
```

Configure image scanning.

```
flux create image repository demoapp \
  --namespace=tgndevs \
  --image=docker.io/tgndevs/demoapp \
  --interval=1m \
  --export > ./clusters/$CLUSTER_NAME/demoapp-registry.yaml
```

Create an `ImagePolicy` to tell Flux which semver range to use when filtering.

```
flux create image policy demoapp \
  --namespace=tgndevs \
  --image-ref=demoapp \
  --select-semver=">=1.0.0" \
  --export > ./clusters/$CLUSTER_NAME/demoapp-policy.yaml
```

Create an `ImageUpdateAutomation` to tell Flux which Git repository to write image updates to:

```
flux create image update demoapp \
  --namespace=tgndevs \
  --git-repo-ref=demoapp \
  --git-repo-path="./manifests" \
  --checkout-branch=main \
  --push-branch=main \
  --author-name=fluxcdbot \
  --author-email=fluxcdbot@users.noreply.github.com \
  --commit-template="{{range .Updated.Images}}{{println .}}{{end}}" \
  --export > ./clusters/$CLUSTER_NAME/demoapp-imageupdate.yaml
```

### Configure notifications

First create a secret with your Slack incoming webhook:

```
kubectl create secret generic slack-url \
  --namespace flux-system \
  --from-literal=address=https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK
```

Create a notification provider for Slack by referencing the above secret.

```
flux create alert-provider demoapp-slack \
  --type slack \
  --channel gitops-demo \
  --secret-ref slack-url \
  --export > ./clusters/$CLUSTER_NAME/demoapp-alert-provider.yaml
```

Create an alert for the demo app. We will get notification every time there's a change or error in any of the app components.

```
flux create alert demoapp \
  --provider-ref demoapp-slack \
  --event-severity info \
  --event-source GitRepository/demoapp \
  --event-source Kustomization/demoapp \
  --export > ./clusters/$CLUSTER_NAME/demoapp-alert.yaml
```
