# Beginner Hands-on

```sh

## Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version --client

## Install K9S
curl -sS https://webinstall.dev/k9s | bash
source ~/.config/envman/PATH.env

## Install Helm
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Subject:

## A simple pod

1. Create a `nginx` pod definition in the `nginx-app.yaml` file, image should be `nginx:1.20`. \
   _Tip: <https://kubernetes.io/docs/concepts/workloads/pods/#using-pods>_
2. Run the pod on the cluster with `kubectl apply -f nginx-app.yaml`
3. Check that the pod is running with `kubectl get po --watch`
4. When the pod gets `Ready`, use `kubectl port-forward` to forward remote port `80` to local port `8080` \
   _Note: `--address='0.0.0.0'` flag can be used in the command to access the port from outside the machine_ \
   _Tip: <https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/#forward-a-local-port-to-a-port-on-the-pod>_
5. Check that NGinx is available with a `curl localhost:8080` or `curl MACHINE_IP:8080`

## Deployment

1. Turn the pod definition into a Deployment definition. \
   _Tip: <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>_
2. Apply the deployment on the cluster
3. Scale the deployment up to 3 replicas and down with `kubectl scale` command \
   _Tip: use the `--help` flag to learn how to use this command_
4. Check the deployment logs with `kubectl logs` command \
   _Tip: use the `--help` flag to learn how to use this command_
5. Configure readiness and liveness probes for the deployment \
   _Tip: <https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/>_

## Networking

1. Declare the container port `80` in the nginx deployment manifest if not already done
2. Define a new service `app` selecting your `nginx` pods \
   _Tip: You can use `---` on a newline to concatenate multiple resources in a single yaml file_ \
   _Tip: <https://kubernetes.io/docs/concepts/services-networking/service/#defining-a-service>_
3. Use `kubectl port-forward` to forward `app` service port `80` to local port `8080` \
   _Note: `--address='0.0.0.0'` flag can be used in the command to access the port from outside the machine_
  
4. Check that NGinx is still available with a `curl localhost:8080` or `curl MACHINE_IP:8080`

## Configuration

1. Create a new ConfigMap named `nginx-config` containing the [index.html](./index.html) file from this directory. \
   _Tip: Check the `kubectl create cm --help` command output for tips on how to create a ConfigMap from a file_ \
   _Tip: <https://kubernetes.io/docs/concepts/configuration/configmap/>_
2. Check that the ConfigMap exists with `kubectl get cm -o yaml`
3. Mount the `index.html` file in the NGinx deployment at `/usr/share/nginx/html`. \
   _Tip: <https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#populate-a-volume-with-data-stored-in-a-configmap>_
4. Add these requests and limits to your pod \
   _Tip: <https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#example-1>_

   ```yaml
       resources:
         requests:
           memory: "64Mi"
           cpu: "250m"
         limits:
           memory: "128Mi"
           cpu: "500m"
   ```

5. Scale your deployment to 100 replicas, what happens ? \
   _Additional documentation for resource management: <https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/>_

## Storage

1. Use the following command to deploy a Local Storage Provisioner:

```sh
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.25/deploy/local-path-storage.yaml
```

2. This will create a storageClass named "local-path" that we will be able to create Persistent Volumes on the cluster node. Use the following command to inspect its content:

```sh
kubectl get storageclass local-path -o yaml
```

3. Create a StatefulSet application that will be using the local-storage StorageClass
   - Each pod volumes must have a storage capacity of `1Gi` and accessModes `ReadWriteOnce`.
   - The pod will use `busybox` as an image and will be mounting the persistent volume in `/mnt` folder.
   - We will use the previously created storage class to provision our volumes.
   - To write some text into the volumes, we will configure the containers to execute the following command on startup: `['sh', '-c', 'echo "The local volume is mounted!" > /mnt/test.txt && sleep 3600']`.

   _Tip: <https://kubernetes.io/blog/2019/04/04/kubernetes-1.14-local-persistent-volumes-ga/#how-to-use-a-local-persistent-volume>_ \
   _Tip: <https://kubernetes.io/fr/docs/concepts/workloads/controllers/statefulset/#composants>_

4. Use kubectl to check the new PV and PVC status \
   _Tip: <https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/>_

5. Check that the pods are running properly

6. Une the `kubectl describe` command on a persistent volume to have a look at the following fields: `Node Affinity` and `Source/Path`

7. Delete the StatefulSet that was created earlier

8. On the host, cat the `test.txt` file that was created into the PV to validate that the data has been persisted

9. Re-create the StatefulSet you just deleted. Are new PV and PVC created ?

## Update

1. Change the nginx image tag to `1.21` for the deployment created earlier \
   _Tip: Check the help with `kubectl set image --help`_
2. Check the update status with `kubectl rollout status ...`
3. Check your deployment history with `kubectl rollout history` command
4. Rollback your changes \
   _Tip: use `kubectl rollout undo`_
5. Change the `background-color` CSS property in the `index.html` file by updating the configMap mounted into the pod.
6. Use `kubectl port-forward` to forward `app` service port `80` to local port `8080` \
   _Note: `--address='0.0.0.0'` flag can be used in the command to access the port from outside the machine_
7. Check if the `index.html` file sourced from the configmap is updated with a `curl localhost:8080` or `curl MACHINE_IP:8080`
8. Rollout the deployment to force pods re-creation and check again if the `index.html` file sourced from the configmap is updated. \
   _Tip: <https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_restart/>_

## Packaging

1. Create an Helm package with `helm create my-app-package` \
   Tip: <https://helm.sh/docs/helm/helm_create/>
2. Browse default Helm chart contents
3. Adapt the Helm chart to your existing resources (nginx deployment, configmap and service). At least the `image` pod property should be sourced from the values file
4. Change the image version and rollback just like you did with plain Kubernetes manifests using the helm commands \
   _Tip: Check out the following commands:_
   - _<https://helm.sh/docs/helm/helm_install/#helm-install>_
   - _<https://helm.sh/docs/helm/helm_upgrade/>_
   - _<https://helm.sh/docs/helm/helm_rollback/>_

## Bonus

1. Add another container alongside nginx inside the pod specification to expose Prometheus metrics (<https://github.com/nginxinc/nginx-prometheus-exporter>). \
   _Tip: There is no official documentation on how to do it with Kubernetes. Although, there is one for Docker: <https://github.com/nginxinc/nginx-prometheus-exporter?tab=readme-ov-file#running-the-exporter-in-a-docker-container>._ \
   _Tip: Have a look at the content of the `docker run` command to deduce the information you need to write in the Kubernetes manifest._ \
   _Tip: Check how to define an environment variables in a container: <https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/#define-an-environment-variable-for-a-container>_ \
   _Tip: Enable NGINX Stub Status page: <https://docs.nginx.com/nginx-amplify/nginx-amplify-agent/configuring-metric-collection/>_ \
   _Tip: Mount configmap keys at different locations: <https://kubernetes.io/docs/concepts/storage/volumes/#using-subpath>_
2. Do the same packaging with the built in kustomize tool from `kubectl` (Documentation: <https://kustomize.io/>)
