# Cluster default ingress controller 
Served via annotation `k8s.ovn.org/open-default-ports`.

For installing the example, please create the resources one by one, as the CUDN will make the namespace and the pod creation to be a bit delayed. 

```bash
# Be located at the default-ingress folder
oc apply -f 1_cudn.yaml

oc apply -f 2_namespace.yaml

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

oc apply -f 2_namespace.yaml

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
# Check how to access the cluster 
oc debug -- curl -s -m 2 -H "Host: ingress.hello.com" $LOAD_BALANCER_IP
```

The latter will show the served content from the Ingress Controller

```
oc debug -- curl -s -m 2 -H "Host: ingress.hello.com" $LOAD_BALANCER_IP
Starting pod/image-debug-qcp6q ...
Served from an isolated Cluster User Defined Network.

Removing debug pod ...
```

## TLS termination
Create an ingress resource which will make use of the standard certificate
```bash
oc apply -f 6_2_ingress-tls-edge.yaml
```

In a debug pod, 
```bash
oc debug
```

create the `ca.crt` file containing the CA used to create the ingress controller certificate

```bash
cat <<EOF > ca.crt
-----BEGIN CERTIFICATE-----
MIIDvzCCAqegAwIBAgIUQIXuftpav/IDCpLjJ70Dhz3jlLQwDQYJKoZIhvcNAQEL
BQAwbzELMAkGA1UEBhMCRVMxDzANBgNVBAgMBkNhcmliZTEQMA4GA1UEBwwHTWFj
b25kbzENMAsGA1UECgwEQUNNRTEMMAoGA1UECwwDQ1NBMSAwHgYDVQQDDBdQcml2
YXRlQ0EgZm9yIENTQSBEZW1vczAeFw0yNDA5MTMxNTU1MTBaFw0yOTA5MTIxNTU1
MTBaMG8xCzAJBgNVBAYTAkVTMQ8wDQYDVQQIDAZDYXJpYmUxEDAOBgNVBAcMB01h
Y29uZG8xDTALBgNVBAoMBEFDTUUxDDAKBgNVBAsMA0NTQTEgMB4GA1UEAwwXUHJp
dmF0ZUNBIGZvciBDU0EgRGVtb3MwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEK
AoIBAQC0te0JhCxqGAbp7loR9uAOu9G1euwvokuWde0tNaYBQv5FDOUsNo05NHTX
6ppA7J4b2p4QllyDxFs5Q9pot13C2n/PwgQK7T0MCPlT5U+Ao+YzEqx+6uCDjb4a
q+hMVDWVoD7UihUHYaDYg525fbCSCBzVAOG2b0H4wJ+VBpNTLiHZPi3sFevRykDD
rToZPj39mVQA8D4dYe8LIKw5FxUKyT3eivRCxDLxtgABFKZWbMPjRAmK5Pd81Gdb
l+eAL35RHzfK2mrwIfIUyEhSSKKzbDOQuBl7GKFPvL5epIX5TeMKSeIgVbn5BN9M
gm8orT4QYchd2ik9cAsdWog/Z5oJAgMBAAGjUzBRMB0GA1UdDgQWBBTs+voWWfDh
z18pmRmVkvbgvK+wOTAfBgNVHSMEGDAWgBTs+voWWfDhz18pmRmVkvbgvK+wOTAP
BgNVHRMBAf8EBTADAQH/MA0GCSqGSIb3DQEBCwUAA4IBAQB88ukRxB9vNu+9fvDl
nUadzG0lLRhlKh114ktx7EX+DjZ2izmncY6P8LZVSNnikoJ54fB1MXyf8CmJ8Eqz
M/2aMav6O0Z2RMWdimU13ZYq4b9i838FwPd9JRjTRU3587BL5pnhdWkFGSPiexxX
diih42JkNwvjv37oUdpxLQzE1tvak+YZX/RWPNtB0C92PDljBeih8CrJEJOafSmf
cbW1XUe4as5t+lZZpiHijHDFTJManAjQPQ396e01RK/pA2G2PV+KNlVtl/7VkAaR
x9OIoMoG/+uUJ+Px/pmxk8SWVHVqPwa3BR8KTkK0ZLbKkgQGxK12DPNeJRNCZl2Y
DREo
-----END CERTIFICATE-----
EOF
```

Still in the debug pod, modify the `/etc/hosts` to assign the haproxy-ingress Load Balancer service External IP to the `hello.apps.meloinvento.com` name.

```bash
echo "__HERE_THE_LOAD_BALANCER_IP__ hello.apps.meloinvento.com"  >> /etc/hosts
```

Check that we can reach the application with HTTPS:

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
