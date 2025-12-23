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
      -n net-demo-ingress hello \
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
# Be located at the isolated folder
oc apply -f 1_cudn.yaml

oc apply -f 2_namespaces.yaml

oc apply -f 3_app.yaml
```
Once we have the apps and their services, we need to setup the prerequisites for the Ingress Controller. We create the service account the IC will run under and the secret to be given as default certificate when SSL termination is required. 

```bash
# Creation of the SA and their privileged SCC assignation
oc apply -f 4_sa.yaml

# Creation of the certificates for the default IC and the pt workload.
oc apply -f ./cert-ingress/cert.yaml
oc apply -f ./cert-passthrough-workload/cert.yaml

helm repo add haproxytech https://haproxytech.github.io/helm-charts
helm repo update
helm install haproxy-ingress haproxytech/kubernetes-ingress \
     -n net-demo-isolated \
     -f ./5_values.yaml

# We create the ingress object via the external name
oc apply -f 6_1_ingress-plain.yaml
```
Now, we can check from a random node, that we are able to reach the ingress controller via the Load Balancer IP. In this example, this has been tested on ARO, so we need to retrieve the Load Balancer IP and we check from a Random node

```bash
LOAD_BALANCER_IP=$(oc get svc \
        -n net-demo-isolated haproxy-ingress \
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

## TLS termination
Create a file `ca.crt`, which is the signing certificate from the `isolated/ingressCA.crt` folder in a debug pod and add a line in the `/etc/hosts` for the `hello.apps.meloinvento.com` host pointing to the HAProxy Load Balancer service. 

```bash
curl --cacert ./ca.crt -v https://hello.apps.meloinvento.com
```

EXAMPE: 
```
sh-5.1# curl --cacert ./ca.crt -v https://hello.apps.meloinvento.com
*   Trying 192.168.126.20:443...
* Connected to hello.apps.meloinvento.com (192.168.126.20) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
*  CAfile: ./ca.crt
* TLSv1.0 (OUT), TLS header, Certificate Status (22):
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.2 (IN), TLS header, Certificate Status (22):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS header, Finished (20):
* TLSv1.2 (IN), TLS header, Unknown (23):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.2 (OUT), TLS header, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS header, Unknown (23):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: C=ES; ST=Caribe; L=Macondo; O=ACME; OU=CSA; CN=CSA Demo
*  start date: Dec 22 13:21:02 2025 GMT
*  expire date: Mar 26 13:21:02 2028 GMT
*  subjectAltName: host "hello.apps.meloinvento.com" matched cert's "*.apps.meloinvento.com"
*  issuer: C=ES; ST=Caribe; L=Macondo; O=ACME; OU=CSA; CN=PrivateCA for CSA Demos
*  SSL certificate verify ok.
* Using HTTP2, server supports multi-use
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* TLSv1.2 (OUT), TLS header, Unknown (23):
* TLSv1.2 (OUT), TLS header, Unknown (23):
* TLSv1.2 (OUT), TLS header, Unknown (23):
* Using Stream ID: 1 (easy handle 0xaaaacb05d2a0)
* TLSv1.2 (OUT), TLS header, Unknown (23):
> GET / HTTP/2
> Host: hello.apps.meloinvento.com
> user-agent: curl/7.76.1
> accept: */*
>
* TLSv1.2 (IN), TLS header, Unknown (23):
* TLSv1.2 (OUT), TLS header, Unknown (23):
* TLSv1.2 (IN), TLS header, Unknown (23):
< HTTP/2 200
< date: Mon, 22 Dec 2025 17:51:07 GMT
< content-length: 54
< content-type: text/plain; charset=utf-8
< alt-svc: h3=":443";ma=60;
<
Served from an isolated Cluster User Defined Network.
* Connection #0 to host hello.apps.meloinvento.com left intact
```

If you don't include the load balancer ip in the /etc/hosts folder, you can opt for the curl sentence: 

```bash
curl --resolve "hello.apps.meloinvento.com:443:<<<LOAD BALANCER IP>>>" --cacert ./ca.crt  -v https://hello.apps.meloinvento.com
```
