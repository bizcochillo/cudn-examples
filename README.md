# Cluster default ingress controller 
Served via annotation `k8s.ovn.org/open-default-ports`.

For installing the example, please create the resources one by one, as the CUDN will make the namespace and the pod creation to be a bit delayed. 

```bash
# Be located at the default-ingress folder
oc apply -f 1_cudn.yaml

oc apply -f 2_namespaces.yaml

oc apply -f 3_netpol.yaml

oc apply -f 4_app.yaml
```

Once installed, we can check if the app is being served with TLS edge termination: 

```bash
curl -s -m 1 \
  https://$(oc get route \
      -n net-demo-ingress hello-openshift \
      -ojsonpath='{.spec.host}')
```

And it will result in the following output
 
```
Served via a default Ingress Controller in the default network!
```

We observe that the pod has an ovn-udn1 interface with an IP given within the range 172.11.0.0/16 as specified in the CUDN CRD

```json
[
  {
    "name": "ovn-kubernetes",
    "interface": "eth0",
    "ips": [
      "10.131.0.125"
    ],
    "mac": "0a:58:0a:83:00:7d",
    "dns": {}
  },
  {
    "name": "ovn-kubernetes",
    "interface": "ovn-udn1",
    "ips": [
      "172.11.1.4"
    ],
    "mac": "0a:58:ac:0b:01:04",
    "default": true,
    "dns": {}
  }
]
```

Also, we can check, that we cannot access the CUDN open ports from the default network, but it's possible to access from the default ingress contoller pods

```bash
# Retrieve the pod IP
POD_IP_DEFAULT_NETWORK=$(oc get pod --selector app=hello-openshift -ojsonpath="{.items[0].status.podIP}")

# It should return an IP within the Pod default overlay network.
echo $POD_IP_DEFAULT_NETWORK

# From the default ingress controller the traffic is allowed
POD_INGRESS_CONTROLLER_DEFAULT=$(oc get pod \
    -n openshift-ingress \
    --selector ingresscontroller.operator.openshift.io/deployment-ingresscontroller=default \
    -ojsonpath='{.items[0].metadata.name}')

oc exec \
    -n openshift-ingress \
    $POD_INGRESS_CONTROLLER_DEFAULT \
    -- curl -m 2 -s http://$POD_IP_DEFAULT_NETWORK:8080

# Not possible to access the app port 8080 via the default network IP from the CUDN
oc debug -n net-demo-ingress -- curl -m 2 -s http://$POD_IP_DEFAULT_NETWORK:8080

# Not possilbe to access the app port 8080 via the default network IP form the default network
oc debug -n default -- curl -m 2 -s http://$POD_IP_DEFAULT_NETWORK:8080
```

# Isolated network with a community custom ingress constroller 

Only as an example, we check the setup with a custom HAProxy ingress controller installed with a Helm Chart. 

```bash
# Be located at the default-ingress folder
oc apply -f 1_cudn.yaml

oc apply -f 2_namespaces.yaml

oc apply -f 3_app.yaml

oc apply -f 4_sa.yaml

helm repo add haproxytech https://haproxytech.github.io/helm-charts
helm repo update
helm install haproxy-ingress haproxytech/kubernetes-ingress \
     -n net-demo-isolated \
     -f ./5_values.yaml \
     -f ./5_values-aro.yaml

# We create the ingress object via the external name
oc apply -f 6_ingress.yaml
```
Now, we can check from a random node, that we are able to reach the ingress controller via the Load Balancer IP. In this example, this has been tested on ARO, so we need to retrieve the Load Balancer IP and we check from a Random node

```bash
LOAD_BALANCER_IP=$(oc get svc \
        -n net-demo-isolated haproxy-ingress-kubernetes-ingress \
        -ojsonpath='{.status.loadBalancer.ingress[0].ip}')

RANDOM_NODE_INDEX=$(( RANDOM % 6 ))
TEST_NODE=$(oc get node \
        -ojsonpath={.items\[$RANDOM_NODE\].metadata.name})

# Check how to access the cluster 
oc debug node/$TEST_NODE \
        -- curl -s -m 2 -H "Host: ingress.hello.com" $LOAD_BALANCER_IP
```

The latter will show the served content from the Ingress Controller

```
(base) ➜  isolated oc debug node/$TEST_NODE \
        -- curl -s -m 2 -H "Host: ingress.hello.com" $LOAD_BALANCER_IP
Temporary namespace openshift-debug-gwrd8 is created for debugging node...
Starting pod/saa-obs-5tvt7-worker-germanywestcentral1-kkfsh-debug-cwxrl ...
To use host binaries, run `chroot /host`. Instead, if you need to access host namespaces, run `nsenter -a -t 1`.
Served from an isolated Cluster User Defined Network.

Removing debug pod ...
Temporary namespace openshift-debug-gwrd8 was removed.
```