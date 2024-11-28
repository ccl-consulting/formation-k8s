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
- run `mv config ~/.kube/`
- Validate your access to the api-server with `kubectl get po`
- Install helm : `curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash`
- Install k9s
```sh
curl -sS https://webinstall.dev/k9s | bash
export PATH="/home/gitpod/.local/bin:$PATH"
```

# Antiaffinity

- Install the  [application](./application/) helm chart on the cluster \
  *Tip: https://helm.sh/docs/helm/helm_install/*
- Prevent multiple pods of your application from running on the same node with an `antiAffinity` of type `requiredDuringSchedulingIgnoredDuringExecution` \
  *Tip: https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#an-example-of-a-pod-that-uses-pod-affinity* \
  *Tip: https://helm.sh/docs/chart_template_guide/named_templates/* \
  *Tip: https://helm.sh/docs/chart_template_guide/control_structures/* \
  *Tip: https://kubernetes.io/docs/reference/labels-annotations-taints/#kubernetesiohostname*
- Scale your deployment to 2, what is the result ? Describe the pod to check the event.
- Change the `antiAffinity` definition to allow multiple pods to run on the same node if there is no other choice \
  *Tip: https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#an-example-of-a-pod-that-uses-pod-affinity*
- Re-deploy the application and validate that both pods can run alongside on the same node

# podDisruptionBudget

- Ensure that your deployment `replicas` are set to 2 in the helm templates
- Create a `podDisruptionBudget` for your deployment with a `maxUnavailable` value set to `1` \
  *Tip: https://kubernetes.io/docs/tasks/run-application/configure-pdb/*
- Delete both pods with `kubectl delete pods -l app.kubernetes.io/name=application`, notice that pods are deleted even though a `podDisruptionBudget` was created. Why ?
- Set the `maxUnavailable` value in the pdb to 0
- Delete both pods with `kubectl delete po -l app.kubernetes.io/name=application`, notice that pods are deleted even though a `podDisruptionBudget` was created. Why ?
- Delete the podDisruptionBudget

# initContainers

- Add an `initContainer` that uses the nginx image and just exit with code 0 to your chart pod template. \
  *Tip: https://kubernetes.io/docs/concepts/workloads/pods/init-containers/#using-init-containers*
- Create a shared emptyDir in the pod, mount in on both containers in `/data`. \
  *Tip: https://kubernetes.io/docs/concepts/storage/volumes/#emptydir*
- Edit the `command` of the initcontainer to generate a certificate in the shared volume with `openssl req -x509 -newkey rsa:4096 -nodes -out /data/cert.pem -keyout /data/key.pem -days 365 -subj '/C=FR/ST=TOULOUSE/L=TOULOUSE/O=Acme Inc. /OU=IT Department/CN=acme.com'`
- Configure nginx to use the generated certificate from the shared volume \
  *Tip: https://nginx.org/en/docs/http/configuring_https_servers.html* \
  *Tip: https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#populate-a-volume-with-data-stored-in-a-configmap* \
  *Tip: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#http-probes*

# Operators

- Install cert-manager on the cluster \
  *Tip: https://cert-manager.io/docs/installation/#default-static-install*
- Check that the Cert-manager CRD were correctly deployed with `kubectl api-resources | grep cert-manager`
- Create a self-signed certificate Issuer \
  *Tip: https://cert-manager.io/docs/configuration/selfsigned/#deployment*
- Use your `self-signed` issuer to sign a certificate \
  *Tip: https://cert-manager.io/v1.1-docs/concepts/certificate/*
- Include the certificate definition in the `application` Helm chart
- Mount the cert-manager generated cert and key on your Nginx Pod (find a way to rename the files to avoid changing the nginx configuration) \
  *Tip: https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/#project-secret-keys-to-specific-file-paths*
- Delete the certificate secret, what happens ?


# User creation

- Create a certificate key for a new user
  ```sh
  openssl genrsa -out developer.key 2048
  ```
- Create a CSR for the user client certificate
  ```sh
  openssl req -new -key developer.key \
    -out developer.csr \
    -subj "/CN=developer"
  ```
- Create the Kubernetes CSR resource from the local CSR file \
  *Tip: https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#normal-user* \
  *Tip: The content of the CSR be converted into a one-liner base64 using the following command: `cat developer.csr | base64 -w0`*
- Apply the CSR resource to the cluster
- Validate that the CSR was created with `kubectl get csr`
- Approve the CSR with `kubectl certificate approve csr_name`
- Retrieve the signed certificate with `kubectl get csr $CSR_NAME -o jsonpath='{.status.certificate}'| base64 -d > developer.crt`
- Add the certificate to your local `kubeconfig` \
  *Tip: https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#add-to-kubeconfig*
- Using the context of your new user, try to run a `kubectl get pods`, can you guess why it is not working ?
- Switch back to `kubernetes-admin@kubernetes` context (hint: use `kubectl config view` to list available context, clusters and users)

# RBAC

- Create a new namespace named `app-dev` and install the `application` chart in it
- Create a new namespace named `app-prod` and install the `application` chart in it

## Application

- Add a service account definition in the `application` chart templates \
  *Tip: https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/*
- Disable default service-account mounting for the deployment in `application` chart \
  *Tip: https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#use-the-default-service-account-to-access-the-api-server*
- Use the service account you created above in the `application` chart pod template. \
  *Tip: https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#manually-create-a-long-lived-api-token-for-a-serviceaccount*

## User

- Create a `viewer` role that can read `deployments`, `replicasets`, `pods`, `services` and `configmaps` \
  *Tip: https://kubernetes.io/docs/reference/access-authn-authz/rbac/#role-and-clusterrole*
- Create a `developer` role that can read and edit `deployments`, `replicasets`, `pods`, `services`, `configmaps` and `secrets` \
  *Tip: https://kubernetes.io/docs/reference/access-authn-authz/authorization/#determine-the-request-verb*

- Modify the `developer` role so that it does not include any permissions for the `secrets` resource if the namespace is `app-prod`. \
  *Tip: https://helm.sh/docs/chart_template_guide/control_structures/* \
  *Tip: Current namespace can be retrieved using `.Release.Namespace`*

- Assign the `developer` roles to your user `developer` \
  *Tip: https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#create-role-and-rolebinding* \
  *Tip: Instead of creating the resource derectly with `kubectl`, it is possible to get the yaml file to be created with `-o yaml --dry-run=client` and then integrate it in the Helm Chart.*

- Using the context of your new user, try to run a `kubectl get pods`

- Edit the developer `Role` in the `app-dev` namespace to allow developer to get pods logs. \
  *Tip: https://kubernetes.io/docs/reference/access-authn-authz/rbac/#referring-to-resources*
- Add a ClusterRole `cluster-user` that can list nodes, bind it to the `developer` user \
  *Tip: https://kubernetes.io/fr/docs/reference/access-authn-authz/rbac/#exemple-de-clusterrole*
- Switch to the developer context, validate that you can list nodes

# NetworkPolicy

- Run a simple network test pod with `kubectl run net-test --image=subfuzion/netcat --command -- /bin/sh -c 'while true; do echo -n read_input | timeout 1 nc -vz 1.1.1.1 53; sleep 2; done'`
- Create a deny-all egress network policy \
  *Tip: https://kubernetes.io/docs/concepts/services-networking/network-policies/#default-deny-all-egress-traffic*
- Check the logs of the `net-test` pod, what changed ?
- Add an exception to allow egress to the address `1.1.1.1` in the NetworkPolicy \
  *Tip: https://kubernetes.io/docs/concepts/services-networking/network-policies/#networkpolicy-resource*
- Create a second network test pod with `kubectl run net-test-1 --image=chainguard/netcat --labels=network=unrestricted --command -- /bin/sh -c 'while true; do echo -n read_input | timeout 1 nc -vz 8.8.8.8 53; sleep 2; done'`
- Check the logs of the `net-test-1` pod
- Edit the network policy to exclude pod `net-test-1` from egress filtering \
  *Tip: https://kubernetes.io/docs/concepts/services-networking/network-policies/#networkpolicy-resource*
- Check the logs of the `net-test-1` pod


# Resources

To go further on pod evictions and guarantees: https://medium.com/container-talks/ultimate-guide-of-pod-eviction-on-kubernetes-588d7f6de8dd
