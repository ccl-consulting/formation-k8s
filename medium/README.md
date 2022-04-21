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
- Set the `SERVER_IP` environment variable: `export SERVER_IP=1.2.3.4` (replace `1.2.3.4` with the IP provided for the lab)
- Download the `kubeconfig` file from the server : `scp -i key ubuntu@${SERVER_IP}:/home/ubuntu/.kube/config ./config`
- Edit the `config` file, and change the `clusters[0].cluster.server` property IP address to the provider lab IP
- run `mkdir ~/.kube`
- run `mv config ~/.kube/`
- Validate your access to the api-server with `kubectl get po`
- Install helm : `curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash`
- Install k9s
```sh
curl -sS https://webinstall.dev/k9s | bash
export PATH="/home/gitpod/.local/bin:$PATH"
```

# Antiaffinity

- Install the `application` helm chart on the cluster
- Prevent multiple pods of your application from running on the same node with an `antiAffinity` of type `requiredDuringSchedulingIgnoredDuringExecution` (https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#an-example-of-a-pod-that-uses-pod-affinity)
- Scale your deployment to 2, what is the result ?
- change the `antiAffinity` definition to allow multiple pods to run on the same node if there is no other choice
- Re-deploy the application and validate that both pods can run alongside on the same node

# PodDistruptionBudget

- Ensure that your deployment `replicas` are set to 2 in the helm templates
- Create a `podDistruptionBudget` for your deployment with a `maxUnavailable` value set to `1` (https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
- delete both pods with `kubectl delete po -l app.kubernetes.io/name=application", what happens ? What events can you notice during deletion ?
- set the `maxUnavailable` value in the pdb to 0
- delete both pods with `kubectl delete po -l app.kubernetes.io/name=application", what happens ? What events can you notice during deletion ?
- Delete the PodDistruptionBudget

# initContainers

- Add an `initContainer` that uses the nginx image and just exit with code 0 to your chart pod template. (https://kubernetes.io/docs/concepts/workloads/pods/init-containers/#using-init-containers)
- Create a shared emptyDir in the pod, mount in on both containers (https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)
- Edit the `command` of the initcontainer to generate a certificate in the shared volume with `openssl req -x509 -newkey rsa:4096 -nodes -out cert.pem -keyout key.pem -days 365`
- Configure nginx to use the generated certificate from the shared volume

# Operators

- Install cert-manager on the cluster (https://cert-manager.io/docs/installation/#default-static-install)
- Check that the Cert-manager CRD were correctly deployed with `kubectl api-resources | grep cert-manager`
- Create a self-signed certificate Issuer (https://cert-manager.io/docs/configuration/selfsigned/#deployment)
- Use your `self-signed` issuer to sign a certificate (https://cert-manager.io/docs/usage/certificate/#creating-certificate-resources)
- Include the certificate definition in the `application` Helm chart
- Mount the cert-manager generated secret on your Nginx Pod (replace the initContainer created one)
- Delete the certificate secret, what happens ?


# User creation

- Create a certificate for a new user
```sh
openssl genrsa -out developer.key 2048
```
- Create a CSR for the user client certificate
```sh
openssl req -new -key developer.key \
  -out developer.csr \
  -subj "/CN=developer"
```
- Create the Kubernetes CSR resource from the local CSR file (https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#create-certificatesigningrequest)
- Apply the CSR resource to the cluster
- Validate that the CSR was created with `kubectl get csr`
- Approve the CSR with `kubectl certificate approve csr_name`
- Retrieve the signed certificate with `kubectl get csr $CSR_NAME -o jsonpath='{.status.certificate}'| base64 -d > developer.crt`
- Add the certificate to your local `kubeconfig` (https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#add-to-kubeconfig)
- Using the context of your new user, try to run a `kubectl get pods`, can you guess why it is not working ?
- Switch back to `kubernetes-admin@kubernetes` context (hint: use `kubectl config view` to list available context, clusters and users)

# RBAC

- Create a new namespace named `app-dev` and install the `application` chart in it
- Create a new namespace named `app-prod` and install the `application` chart in it

## Application

- Add a service account definition in the `application` chart templates
- Disable default service-account mounting for the deployment in `application` chart (https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#use-the-default-service-account-to-access-the-api-server)
- Use the service account you created above in the `application` chart pod template.

## User

- Create a `viewer` role that can read `deployments`, `replicasets`, `pods`, `services` and `configmaps` in both namespaces
- Create a `developer` role that can read and edit `deployments`, `replicasets`, `pods`, `services`, `configmaps` and `secrets` in the `app-dev` namespace.

- Create a `developer` role that can "get", "watch" and "list" `deployments`, `replicasets`, `pods`, `services` and `configmaps` in the `app-prod` namespace.

- Assign the `developer` roles to your user `developer` (https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#create-role-and-rolebinding)

- Using the context of your new user, try to run a `kubectl get pods`

- Edit the developer `Role` in the `app-dev` namespace to allow developer to get pods logs.
- Add a ClusterRole `cluster-user` that can list nodes, bind it to the `developer` user
- Switch to the developer context, validate that you can list nodes

# NetworkPolicy


- Run a simple network test pod with `kubectl run -test --image=nicolaka/netshoot --  bash -c 'while true; do drill google.fr; sleep 2; done'`
- Create a deny-all egress network policy (https://kubernetes.io/docs/concepts/services-networking/network-policies/#default-deny-all-egress-traffic)
- Check the logs of the `net-test` pod, what changed ?
- Add an exception to allow egress to the address `8.8.8.8` in the NetworkPolicy
- Create a second network test pod with `kubectl run net-test-1  --image=nicolaka/netshoot -l network=unresricted --  bash -c 'while true; do drill google.fr; sleep 2; done'`
- Edit the network policy to exclude pod `net-test-1` from egress filtering
- Execute an interactive session on both pods to check that traffic is correctly filtered (with `kubectl exec -it pod_name -- bash`)



# Resources

To go further on pod evictions and guarantees: https://medium.com/container-talks/ultimate-guide-of-pod-eviction-on-kubernetes-588d7f6de8dd
