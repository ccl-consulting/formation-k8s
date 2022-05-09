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




# Istio

- Create the sample Istio application using `kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.13/samples/bookinfo/platform/kube/bookinfo.yaml`
- Wait until pods are deployed, then validate the application is correctly working with `kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"`

> Note: response should be `<title>Simple Bookstore App</title>`

# Istio gateway

- Create a certificate for your istio gateway for domain "*.test.mock" using cert-manager
- Create an Istio ingress gateway for your mesg, responding to HTTP trafic on host "*.test.mock"

> NOTE: The gateway shoud use this selector :
```
    selector:
      istio: ingressgateway
```

- Setup HTTP to HTTPS redirection on your gateway

# Istio VirtualService

- Create a DestinationRoute that route traffix for "product.default.svc.cluster.local" with a single "v1" subset that filter workload with "version=v1" label (https://istio.io/latest/docs/reference/config/networking/destination-rule/)
- Create a VirtualService that route all your traffic from the gateway on "product.test.mock" to the product service and attach the "v1" subset

- Get the ingress public service port : `export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')`
- Adapt this command with your public server ip : `export INGRESS_HOST="<SERVER-IP>"`
- run `export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT`
- Ensure that you can connect to the service using `curl http://$GATEWAY_URL/productpage --header "Host: product.test.mock" | grep -o "<title>.*</title>"`


# Visualization

- get Kiali web ui node port with `export KIALI_PORT=$(kubectl -n istio-system get service kiali -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')`
- get the complete kiali URL using:

```
export KIALI_URL="http://$INGRESS_HOST:$KIALI_PORT"
echo $KIALI_URL
```

- connect to the kiali interface with a browser at `$KIALI_URL`
- Start to generate load on your service with `while true; do curl http://$GATEWAY_URL/productpage --silent -o /dev/null --header "Host: product.test.mock";  echo "OK"; sleep 0.1; done` in your terminal
- Select default namespace for observations in kiali
- Wait for one minute and explore results and metrics from graph view


# Progressive deployment

- Create a new destination route for the `reviews` service. Host should be set to the service name and there must be three subsets `v1`, `v2` and `v3` that matches the label `version` of the deployments.
- Create a new VirtualService for `reviews`, it should route all requests for `reviews` service to the `v1` subset
- Wait for one minute and explore results and metrics from graph view, what can you observe ?
- Progressivly migrate you traffic percentage to the v3 service by editing your VirtualService (https://istio.io/latest/docs/tasks/traffic-management/traffic-shifting/)
- Wait for one minute and explore results and metrics from graph view, what can you observe ?

# Circuit breaker

- Stop your curl loop
- Run a load generation pod with `kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.13/samples/httpbin/sample-client/fortio-deploy.yaml`
- Get the name of the pod into an environment variable with `export FORTIO_POD=$(kubectl get pods -l app=fortio -o 'jsonpath={.items[0].metadata.name}')`
- Ensure that connection can be established with `kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio curl  -H "Host: product.test.mock" "$GATEWAY_URL/productpage"`, you should see an HTML page with a status 200.
- Edit the `reviews` DestinationRule to add a circuit breaker that triggers when `http1MaxPendingRequests` is above one and allow all pods to be ejected. (https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/#configuring-the-circuit-breaker)
- Start load generation with `kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -H "Host: product.test.mock" -c 3 -qps 0 -n 0 -t -1s -loglevel Warning "$GATEWAY_URL/productpage"`
- Go on the `Service` tab in Kiali
- Click on the `reviews` service
- Wait for some time, what can you observe ?
- Remove the circuit breaker from the definition, look for changes of the `reviews` service in Kiali interface

# Mirroring

- Edit the definition of your `reviews` VirtualService to mirror `100%` of the traffic going to the `v3` subset to the `v2` subset
- Wait some time on the `reviews` service interface, what can you observe ?
