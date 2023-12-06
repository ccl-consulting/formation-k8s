


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


# A simple pod

1. Create a `nginx` pod definition in the `nginx-app.yaml` file, image should be `nginx:1.20`. (tip: https://kubernetes.io/docs/concepts/workloads/pods/#using-pods)
2. Run the pod on the cluster with `kubectl apply -f nginx-app.yaml`
3. Check that the pod is running with `kubectl get po --watch`
4. When the pod gets `Ready`, use `kubectl port-forward` to forward remote port `80` to local port `8080`
5. Check that NGinx is available with a `curl localhost:8080`
6. Note: `--address='0.0.0.0'` flag can be used in the command to access the port from outside the machine

# Deployment

1. Turn the pod definition into a Deployment definition. (tip: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
2. Apply the deployment on the cluster
3. scale the deployment up to 3 replicas and down with `kubectl scale` command
4. Check the deployment logs with `kubectl logs` command
5. Configure readiness and liveness probes for the deployment (tip: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)

# Networking

1. Declare the pod port `80` in the deployment manifest if not already done
2. Define a new service `app` selecting your `nginx` pods (You can use `---` on a newline to concatenate multiple resources in a single yaml file !). (tip: https://kubernetes.io/docs/concepts/services-networking/service/#defining-a-service)
3. Use `kubectl port-forward` to forward `app` service port `80` to local port `8080`
4. Check that NGinx is still available with a `curl localhost:8080`

# Configuration

1. Create a new ConfigMap containing the `index.html` file from this directory. (Check the `kubectl create cm --help` command output for tips on how to create a ConfigMap from a file) (Check the `kubectl create cm --help` command output for tips on how to create a ConfigMap from a file). (tip: https://kubernetes.io/docs/concepts/configuration/configmap/)
2. Check that the ConfigMap exists with `kubectl get cm -o yaml`
3. Mount the `index.html` file in the NGinx deployment at `/usr/share/nginx/html`. (tip: https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#populate-a-volume-with-data-stored-in-a-configmap)
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
5. Scale your deployment to 100 replicas, what happens ? (additional documentation for resource management: https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

# Storage

1. Use the following command to deploy a Local Storage Provisioner:

   ```
   kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.25/deploy/local-path-storage.yaml
   ```

2. This will create a storageClass named "local-path" that we will be able to create Persistent Volumes on the cluster node. Use the following command to inspect its content:

   ```
   kubectl get storageclass local-path -o yaml
   ```

3. Create a StatefulSet application that will be using the local-storage StorageClass (tip: [https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/#create-a-persistentvolume](https://kubernetes.io/blog/2019/04/04/kubernetes-1.14-local-persistent-volumes-ga/#how-to-use-a-local-persistent-volume))

   - Each pod volumes must have a storage capacity of `1Gi` and accessModes `ReadWriteOnce`.
   - The pod will use `busybox` as an image and will be mounting the persistent volume in `/mnt` folder.
   - We will use the previously created storage class to provision our volumes.
   - To write some text into the volumes, we will configure the containers to execute the following command on startup: `['sh', '-c', 'echo "The local volume is mounted!" > /mnt/test.txt && sleep 3600']`.

   <details><summary>show answer</summary>
   <p>
   
   ```
   cat << EOF | kubectl apply -f -
   apiVersion: apps/v1
   kind: StatefulSet
   metadata:
     name: my-app-with-local-storage
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: my-app-with-local-storage
     template:
       metadata:
         labels:
           app: my-app-with-local-storage
       spec:
         containers:
         - name: my-container
           image: registry.k8s.io/busybox
           command:
           - "/bin/sh"
           args:
           - "-c"
           - 'echo "The local volume is mounted!" > /mnt/test.txt && sleep 3600'
           volumeMounts:
           - name: my-volume
             mountPath: /mnt
     volumeClaimTemplates:
     - metadata:
         name: my-volume
       spec:
         accessModes: [ "ReadWriteOnce" ]
         storageClassName: "local-path"
         resources:
           requests:
             storage: 1Gi
   EOF
   ```

   </p>
   </details>

4. Use kubectl to check the new PV and PVC status

5. Check that the pods are running properly

6. Describe one of the PV and take a look at the following fields: `Node Affinity` and `Source Path`

7. Delete the StatefulSet that was created earlier

8. On the host, cat the `test.txt` file that was created into the PV to validate that the data has been persisted

9. Re-create the StatefulSet you just deleted. Are new PV and PVC created ?

   <details><summary>show answer</summary>
   <p>
       
       No. the .spec.volumeClaimTemplates field in the STS sets a fixed name for the PVC which will result in the previously created ones to be used.
   </p>
   </details>

# Update

1. Change the nginx image tag to `1.21` (tip: Check the help page of `kubectl set image`)
2. Check the update status with `kubectl rollout status ...`
3. Check your deployment history with `kubectl rollout history` command
4. Rollback your changes (tip: use `kubectl rollout undo`)
5. Change the `background-color` CSS property in the `index.html` file, create a new configmap with the updated

# Packaging

1. Create an Helm package with `helm create my-app-package` (tip: https://helm.sh/docs/helm/helm_create/)
2. Browse default Helm chart contents
3. Adapt the Helm chart to your existing manifests, at least the `image` pod property should be sourced from values file
4. Change the image version and rollback just like you did with plain Kubernetes manifests using the helm command (tip: check out the `install`, `upgrade` and `rollback` commands help pages)


# Bonus

1. Add a monitoring container alongside nginx in the `app` pod to expose Prometheus metrics (https://github.com/nginxinc/nginx-prometheus-exporter).
2. Do the same packaging with the built in kustomize tool from `kubectl` (Documentation: https://kustomize.io/)
