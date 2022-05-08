# Prerequisites

## Work from Gitpod

- Once connected to gitpod, open a terminal
- install kubectl:

```sh
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin
```

- Create a file named `key.pem`, paste the contents of the provider private key
- run `chmod 600 key.pem`
- Set the `SERVER_IP` environment variable: `export SERVER_IP=1.2.3.4` (replace `1.2.3.4` with the IP provided for the lab)
- Download the `kubeconfig` file from the server : `scp -i key.pem ubuntu@${SERVER_IP}:/home/ubuntu/.kube/config ./config`
- Edit the `config` file, and change the `clusters[0].cluster.server` property IP address to the provider lab IP
- run `mkdir ~/.kube`
- run `cp config ~/.kube/`
- Validate your access to the api-server with `kubectl get po`
- Install helm : `curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash`
- Install k9s
```sh
curl -sS https://webinstall.dev/k9s | bash
export PATH="/home/gitpod/.local/bin:$PATH"
```


# HPA

- Apply the resource consuming application deployment from `php-deployment.yaml`
- Once the pod is up, ensure that you can see it's metrics using `kubectl top pod`
- Add the HPA definition to the deployment file. (hint: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/#creating-the-autoscaler-declaratively)
- Increase the load on the service and observe scaling with `k9s` : https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/#increase-load
- Stop the burst once the HPA is stabilized. Wait for the pods to scale down
- Limit scale down to one pod per minute (hint: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#configurable-scaling-behavior)
- Increase the load on the service and observe scaling with `k9s` : https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/#increase-load
- Stop the burst once the HPA is stabilized. Wait for the pods to scale down
- Delete everything with `kubectl delete -f php-deployment.yaml`

# Security policies


- Add user warning on the restricted policy for new pods using `kubectl label namespace default pod-security.kubernetes.io/warn=restricted`
- Try to run a privileged pod with `kubectl run --rm -i --tty busybox --image=busybox --restart=Never --overrides='{"spec": {"template": {"spec": {"containers": [{"securityContext": {"privileged": true} }]}}}}' -- whoami`, what can you observe ?

- Remove the warning and replace it with a strict enforcment

```sh
kubectl label namespace default pod-security.kubernetes.io/warn-
kubectl label namespace default pod-security.kubernetes.io/enforce=restricted
```
- Try to run a privileged pod with `kubectl run --rm -i --tty busybox --image=busybox --restart=Never --overrides='{"spec": {"template": {"spec": {"containers": [{"securityContext": {"privileged": true} }]}}}}' -- whoami`, what can you observe ?

- Adapt the `webserver.yaml` until you can safely deploy it on the cluster

> NOTE: The deployment will still be created, but replicaset wont be able to instanciate pods, you can have a look at the replicaset events for more informations about why and how to fix it.

- remove the webserver with `kubectl delete -f webserver.yaml`
- remove the security policy with `kubectl label namespace default pod-security.kubernetes.io/enforce-`

# Taints

- Add a taint `team=product:PreferNoSchedule` to your kubernetes node
- Install webserver with `kubectl apply -f webserver.yaml`, what can you observe ?
- Replace the node taint to evict workload that does not tolerate the taint, ensure that the webserver is evicted properly (hint: https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- Replace the node taint to strictly forbid scheduling workload that does not tolerate the taint, what can you observe ?
- Create the toleration to allow pod scheduling on your node

# Bonus : OPA

> NOTE: If you have not finished the exercices from the intermediate session, do them them before this bonus.

- Install and configure OPA Gatekeeper on the cluster : https://www.openpolicyagent.org/docs/latest/kubernetes-tutorial/
- Create a rego policy to prevent image pull from untrusted registries (use a dummy domain as whitelisted registry)
- Apply the policy to your cluster
- Try to deploy an image from Dockerhub, observe the result.

