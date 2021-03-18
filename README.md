# gitops-demo


## Set up Flux

Configuration steps for Flux CD

Install Flux CD.

```shell
curl -s https://toolkit.fluxcd.io/install.sh | sudo bash
```

Validate Flux pre-requisites.

```shell
flux check --pre
```

Export your GitHub username and Personal Access Token with full access to the repo scope.

```shell
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```

Bootstrap the Flux system components.

```shell
flux bootstrap github \
  --owner=tgndevs \
  --repository=gitops-demo \
  --branch=main \
  --private=true \
  --path=./clusters/GitopsAks
```

Check that Flux components have been installed correctly.

```shell
flux check
```

## Set up application

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
  --export > ./clusters/GitopsAks/demoapp-source.yaml
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
  --export > ./clusters/GitopsAks/demoapp-kustomization.yaml
```

After a few moments you should see how your Kustomization has been applied and it's ready and the app should start to be deployed.

```
$ kubectl -n fineract  get kustomizations
NAME       READY   STATUS                                                            AGE
fineract   True    Applied revision: main/50492cfbb8578662839b5fae54b0166da4b592a6   3h33m
```