


```sh

## Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

## Install K9S
curl -sS https://webinstall.dev/k9s | bash

## Install Helm
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```


Subject:


# A simple pod

1. Create a `nginx` pod definition in the `nginx-app.yaml` file, image should be `nginx:1.20`. (tip: https://kubernetes.io/docs/concepts/workloads/pods/#using-pods)
2. Run the pod on the cluster with `kubectl apply -f nginx-app.yaml`
3. Check that the pod is running with `kubectl get po --watch`
4. When the pod gets `Ready`, use `kubectl port-forward` to forward remote port `80` to local port `8080`
5. Check that NGinx is available with a `curl localhost:8080`

# Deployment

1. Turn the pod definition into a Deployment definition
2. Apply the deployment on the cluster
3. scale the deployment up to 3 replicas and down with `kubectl scale` command
4. Check the deployment logs with `kubectl logs`
5. Configure readiness and liveness probes for the deployment

# Networking

1. Expose the pod port `80` in the manifest
2. Add label selectors
3. Define a new service `app` selecting your `nginx` pods (You can use `---` on a newline to concatenate multiple resources in a single yaml file !)
4. Use `kubectl port-forward` to forward `app` service port `80` to local port `8080`
5. Check that NGinx is still available with a `curl localhost:8080`

# Configuration

1. Create a new ConfigMap containing the `index.html` file from this directory (Check the `kubectl create cm --help` command output for tips on how to create a ConfigMap from a file)
2. Check that the ConfigMap exists with `kubectl get cm -o yaml`
3. Mount the `index.html` file in the NGinx deployment at `/usr/share/nginx/html`
4. Add these requests and limits to your pod
```yaml
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```
5. Scale your deployment to 100 replicas, what happens ?

# Update

1. Change the nginx image tag to `1.21` (tip: Check the help page of `kubectl set image`)
2. Check the update status with `kubectl rollout status ...`
3. Check your deployment history with `kubectl rollout history` command
4. Rollback your changes (tip: use `kubectl rollout undo`)
5. Change the `background-color` CSS property in the `index.html` file, create a new configmap with the updated

# Packaging

1. Create an Helm package with `helm create my-app-package`
2. Browse default Helm chart contents
3. Adapt the Helm chart to your existing manifests, at least the `image` pod property should be sourced from values file
4. Change the image version and rollback just like you did with plain Kubernetes manifests using the helm command (tip: check out the `install`, `upgrade` and `rollback` commands help pages)


# Bonus

1. Add a monitoring container alongside nginx in the `app` pod to expose Prometheus metrics (https://github.com/nginxinc/nginx-prometheus-exporter)
2. Do the same packaging with the built in kustomize tool from `kubectl` (Documentation: https://kustomize.io/)
