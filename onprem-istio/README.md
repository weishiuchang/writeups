This documents my trials and tribulations with learning [Istio 1.5](https://istio.io)

I decided to figure out what is this Istio mesh that everyone's talking about. Coming from a background of bastic Kubernetes with Traefik as the Ingress controller I will be trying to relate all the new resources and mesh terms with my existing understanding of data flows, so this write-up will have sprinklings of (hopefully) eureka notes and "oh that's what it gives me."

# Sandbox

* [k3d version v1.7.0](https://github.com/rancher/k3d)
* [k3s version v1.17.3-k3s1](https://k3s.io)
* [istio/istioctl version: 1.5.1](https://istio.io)

## Install k3d with some storage allocated to local-path-provisioner

```bash
# k3d create --volume localstore:/var/lib/rancher/k3s/storage --publish 443:443 --publish 80:80 --server-arg '--no-deploy=servicelb' --server-arg '--no-deploy=traefik' --enable-registry --workers 5
# k3d get-kubeconfig
```

* I like [MetalLB](https://github.com/metallb/metallb), and it does lend a certain sense of realism and familiarity to what I should see for an on-prem installation, so with that flimsy logic I went ahead and disabled the deployment of the k3s built-in lb with '--no-deploy=servicelb'
* Since I will be using Istio as the Ingress Controller, disabling the built-in Traefik is also just as straightforward with '--no-deploy=traefik'

```bash
# cat > ./metallb-values <<EOF
configInline:
  address-pools:
    - name: dynamic
      protocol: layer2
      addresses:
        - 172.18.100.0/30
EOF
# helm upgrade --install --debug --atomic -n kube-system metallb stable/metallb -f ./metallb-values.yaml
```

In case you were wondering where I got the `172.18.100.0/30` subnet, it's a range that falls under the docker0 `172.17.0.0/16` subnet range and is *near* the k3s node ips `kubectl get nodes -o wide`

## Install istio from istioctl

Istioctl can be downloaded from their docker image (I'm sure there are easier ways, but this was easy for me):

```bash
# docker create istio/istioctl:1.5.1
# docker ps -a

CONTAINER ID        IMAGE                      COMMAND                  CREATED             STATUS              PORTS                                                              NAMES
f23bc253c2d8        istio/istioctl:1.5.1       "/usr/local/bin/isti…"   9 seconds ago       Created                                                                                eloquent_euler

# docker cp f23:/usr/local/bin/istioctl ~/.local/bin/.
# istioctl manifest apply --set profile=demo
```

# Let's Play

New words! Gateways, VirtualServices, DestinationRules, ServicesEntries, human sacrifice, dogs and cats living together, *mass hysteria*! No really. What I expected of an Ingress controller replacement is an edge service with an allocated loadbalanced IP, and the most straightforward way to find that is with a services grep:

```bash
# kubectl get services -A | egrep -i 'type|loadbalancer'
NAMESPACE      NAME                        TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)
istio-system   istio-ingressgateway        LoadBalancer   10.43.229.84    172.18.100.0   15020:31477/TCP,80:...
```

Looks like our [MetalLB](https://github.com/metallb/metallb) assigned `172.18.100.0` to this thing called `istio-ingressgateway`. Well that's a fair bet it's what we were looking for. **But**, it doesn't seem to respond on http:

```bash
# curl -v http://localhost
*   Trying ::1:80...
* TCP_NODELAY set
* Connected to localhost (::1) port 80 (#0)
> GET / HTTP/1.1
> Host: localhost
> User-Agent: curl/7.66.0
> Accept: */*
>
* Recv failure: Connection reset by peer
* Closing connection 0
curl: (56) Recv failure: Connection reset by peer
```

Well truthfully that's not that surprising actually as we haven't given it anything to forward to. So let's do what would have come naturally and create a service with an Ingress:

```bash
# kubectl create deployment helloworld --image gcr.io/google-samples/hello-app:1.0
# kubectl expose deployment helloworld --port 80 --target-port 8080
# kubectl apply -f - <<EOF
kind: Ingress
apiVersion: extensions/v1beta1
metadata:
  name: helloworld
spec:
  rules:
    - host: helloworld.fireflyclass.com
      http:
        paths:
          - backend:
              serviceName: helloworld
              servicePort: 80
            path: /
EOF
# echo '172.18.100.0 helloworld.fireflyclass.com' >> /etc/hosts
```

We created a Deployment using Google's simple webpage that just returns some plain text and listens on 8080. The Service exposes it as port 80 while the Ingress maps the hostname `helloworld.fireflyclass.com` to that backend Service. The `/etc/hosts` just gives us a pseudo-dns lookup for `helloworld.fireflyclass.com`

```bash
# curl -v http://helloworld.fireflyclass.com
*   Trying 172.18.100.0:80...
* TCP_NODELAY set
* Connected to helloworld.fireflyclass.com (172.18.100.0) port 80 (#0)
> GET / HTTP/1.1
> Host: helloworld.fireflyclass.com
> User-Agent: curl/7.66.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 404 Not Found
< date: Sat, 18 Apr 2020 13:30:41 GMT
< server: istio-envoy
< content-length: 0
<
* Connection #0 to host helloworld.fireflyclass.com left intact
```

Hm. Well it certainly looks like Istio has done away with supporting Ingresses, so what is the "Istio-way" to expose a Service? It looks like they've created a couple of **C**ustom **R**esource **D**efinitions: Gateways and VirtualServices. So let's get rid of our Ingress and examine the Gateway that came deployed with Istio:

```bash
# kubectl delete ingress helloworld
# kubectl get -n istio-system gateway ingressgateway -o yaml
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"networking.istio.io/v1alpha3","kind":"Gateway","metadata":{"annotations":{},"labels":{"operator.istio.io/component":"IngressGateways","operator.istio.io/managed":"Reconcile","operator.istio.io/versio
n":"1.5.1","release":"istio"},"name":"ingressgateway","namespace":"istio-system"},"spec":{"selector":{"istio":"ingressgateway"},"servers":[{"hosts":["*"],"port":{"name":"http","number":80,"protocol":"HTTP"}}]}}
  creationTimestamp: "2020-04-18T13:09:43Z"
  generation: 1
  labels:
    operator.istio.io/component: IngressGateways
    operator.istio.io/managed: Reconcile
    operator.istio.io/version: 1.5.1
    release: istio
  name: ingressgateway
  namespace: istio-system
  resourceVersion: "1103"
  selfLink: /apis/networking.istio.io/v1beta1/namespaces/istio-system/gateways/ingressgateway
  uid: 21269be3-9183-4c3b-8d69-c3b49f2214dd
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - '*'
    port:
      name: http
      number: 80
      protocol: HTTP
```

`spec.servers.hosts` is currently set for splat hosts, so according to Istio docs on [gateways](https://istio.io/docs/reference/config/networking/gateway/) its purpose is to expose ports. Well it has done so on port 80. Now then, how do we associate Services with it?

[VirtualServices](https://istio.io/docs/reference/config/networking/virtual-service/) appears to be the answer. I say *appears* because I keep coming across far richer content swirling around [DestinationRules](https://istio.io/docs/reference/config/networking/destination-rule/) and [ServiceEntry](https://istio.io/docs/reference/config/networking/service-entry/), which for someone like me who's just trying to associate what I knew of basic Kubernetes Ingress network flows with Mesh networking, does nothing but confuse the hell out of me. But hey, that's why I'm down this twisted path to untangle some of these technologies and document what I have learned.

So let's create a VirtualService for our helloworld Service:

```bash
# kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: helloworld
spec:
  hosts:
  - "helloworld.fireflyclass.com"
  gateways:
  - ingressgateway
  http:
    - route:
      - destination:
          host: helloworld.default.svc.cluster.local
          port:
            number: 80
EOF
```

**Wait, why is curl still 404'ing?**

```bash
# curl -v http://helloworld.fireflyclass.com
*   Trying 172.18.100.0:80...
* TCP_NODELAY set
* Connected to helloworld.fireflyclass.com (172.18.100.0) port 80 (#0)
> GET / HTTP/1.1
> Host: helloworld.fireflyclass.com
> User-Agent: curl/7.66.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 404 Not Found
< date: Sat, 18 Apr 2020 13:52:55 GMT
< server: istio-envoy
< content-length: 0
<
* Connection #0 to host helloworld.fireflyclass.com left intact
```

Well after some googling it turns out that Gateway specifications in VirtualServices need to be namespace qualified! Make sense, but certainly a divergence from the singleton world of traditional Kubernetes Ingress Controllers:

```bash
# kubectl patch virtualservice helloworld --type merge -p $'{"spec":{"gateways":["istio-system/ingressgateway"]}}'
```

Speaking of implicit understandings - I have created the helloworld Deployment, Service, and VirtualService in the Default namespace.

So let's try again, this time with feeling:

```bash
# curl http://helloworld.fireflyclass.com
Hello, world!
Version: 1.0.0
Hostname: helloworld-7859b66cdf-v2wcc
```

Eureka! We have **at least** been able to expose a singular service! On insecured http, using [VirtualService](https://istio.io/docs/reference/config/networking/virtual-service/) instead of [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/), and without any of the [benefits](https://istio.io/docs/reference/config/networking/sidecar/) of a mesh network. With this baseline we can now start to build upon what we know and begin pushing into the complexities of Istio.

## TLS

First things first, let's setup an HTTP**S** termination. This is important.

Well actually, the *first* first thing is to create a couple of certs: A CA and a splat cert for `*.fireflyclass.com`

```bash
# cat > ca.conf <<EOF
[ req ]
prompt             = no
x509_extensions    = v3_req
default_bits       = 2048
default_md         = sha256
distinguished_name = req_distinguished_name

[ req_distinguished_name ]
countryName         = US
stateOrProvinceName = Colorado
localityName        = Boulder
organizationName    = fireflyclass.com
commonName          = ca.fireflyclass.com

[ v3_req ]
basicConstraints       = critical, CA:TRUE
# keyCertSign is required for this to be a true CA
keyUsage               = critical, keyCertSign, cRLSign
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid:always,issuer
EOF
# openssl req -x509 -newkey rsa:2048 -config ./ca.conf -extensions v3_req -days 3650 -nodes -keyout ./ca.key -out ./ca.crt
```

```bash
# cat > fireflyclass.conf <<EOF
[ req ]
prompt             = no
x509_extensions    = v3_req
default_bits       = 2048
default_md         = sha256
distinguished_name = req_distinguished_name

[ req_distinguished_name ]
countryName                = US
stateOrProvinceName        = Colorado
localityName               = Boulder
organizationName           = FireflyClass
commonName                 = *.fireflyclass.com

[ v3_req ]
basicConstraints       = CA:FALSE
keyUsage               = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectKeyIdentifier   = hash
authorityKeyIdentifier = keyid:always,issuer
extendedKeyUsage       = serverAuth
subjectAltName         = @alt_names

[alt_names]
# redundent - but required to make wildcard certs work
DNS.1   = *.fireflyclass.com
DNS.2   = fireflyclass.com
EOF
# openssl req -newkey rsa:2048 -config ./fireflyclass.conf -extensions v3_req -nodes -keyout ./fireflyclass.key -out ./fireflyclass.csr
```

```bash
# openssl x509 -req -in ./fireflyclass.csr -CA ./ca.crt -CAkey ./ca.key -CAcreateserial -out ./fireflyclass.crt -days 3650 -sha256 -extensions v3_req -extfile ./fireflyclass.conf
```

And apparently to setup a TLS Ingress, it is as simple as setting up another Gateway:

```bash
# kubectl create -n istio-system secret generic fireflyclass-credential --from-file=cert=./fireflyclass.crt --from-file=key=./fireflyclass.key
# kubectl create -n istio-system secret generic fireflyclass-credential-cacert --from-file=cacert=./fireflyclass.crt
# kubectl apply -f - <<EOF
kind: Gateway
apiVersion: networking.istio.io/v1beta1

metadata:
  name: ingressgateway-tls
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway # use istio ingress Service
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      httpsRedirect: true
      credentialName: fireflyclass-credential
    hosts:
      - "*"
EOF
```

I have been reliably [told](https://istio.io/docs/tasks/traffic-management/ingress/secure-ingress-sds/#configure-a-mutual-tls-ingress-gateway) that the CA Secret name must end in `-cacert`. Fair enough. Now let's modify our helloworld VirtualService to be accessible from our new TLS Gateway:

```bash
# kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: helloworld
spec:
  hosts:
  - "helloworld.fireflyclass.com"
  gateways:
  - istio-system/ingressgateway
  - istio-system/ingressgateway-tls
  http:
    - route:
      - destination:
          host: helloworld.default.svc.cluster.local
          port:
            number: 80
EOF
```

The important bit here is the addition of our ingressgateway-tls to the list of Gateways. Curl should fail without trusting our CA, and succeed with our CA:

```bash
# curl https://helloworld.fireflyclass.com
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

```bash
# curl --cacert ./ca.crt https://helloworld.fireflyclass.com
Hello, world!
Version: 1.0.0
Hostname: helloworld-7859b66cdf-v2wcc
```

Wonderful! We have a TLS Ingress replacement of our Traefik Controller! It's now cemented in our minds that VirtualService == Ingress.

# Sidecars

Now here's something interesting. We don't have [sidecars](https://istio.io/docs/reference/config/networking/sidecar/) in our helloworld pods.

```bash
# kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
helloworld-7859b66cdf-v2wcc   1/1     Running   0          100m
```

Containers is 1 of 1. According to Istio docs this means that service is not part of the mesh network. **But!** It is still reachable! That's good, that allows us an avenue for migrating services into the cluster and more importantly a fall back if we absolutely must get the Service out of the mesh and working right away as part of troubleshooting.

This was a *eureka* moment for me. I wondered why have a sidecar if Services still work with Gateways and VirtualServices, and it is because the sidecar allows us to now apply rules and policies like mTLS, traffic shaping, etc.

Let us add our helloworld Deployment into the Istio mesh:

```bash
# istioctl kube-inject -f <(kubectl get deployment helloworld -o yaml) | kubectl apply -f -
# kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
helloworld-695c77f7bd-wptgw   2/2     Running   0          32s
```

The important bit here is the 2 of 2 containers in the pod, one of which is the injected sidecar from Istio.

# mTLS

Of all the features afforded by Istio mesh, what interest me next is the [auto mutual TLS feature](https://istio.io/pt-br/docs/tasks/security/authentication/auto-mtls/) for all sidecar-services. This seems like a natural extension of the TLS Ingress Gateway from earlier as it essentally turns all TCP connections between every Service into encrypted links.

To keep us from having to create numerous DestinationRules, an automatic mTLS setting at the Istio level can create it all for us behind the scenes:

```bash
# istioctl manifest apply --set profile=demo --set values.global.mtls.auto=true --set values.global.mtls.enabled=false
```

This enables mtls in the cluster, but with a default policy of `permissive`. Which should mean both cleartext and TLS protected communications to our helloworld Service are allowed. And according to Istio [FAQ](https://istio.io/faq/security/#verify-mtls-encryption), we can check for encrypted links with tcpdump. Well, I won't go that far with this write up, but I will toggle our mesh default policy to *STRICT* mode so we get denied communications from other pods not in our mesh network. Oh! I also learned that policies are [applied](https://www.arctiq.ca/our-blog/2020/3/12/authentication-policy-and-auto-mtls-in-istio-1-5/) at the mesh level, at the namespace level, at the pod level, and at the port level, with the more specific levels winning.

```bash
# kubectl run curltest -it --rm --image infoblox/dnstools
# curl http://helloworld.default.svc.cluster.local
Hello, world!
Version: 1.0.0
Hostname: helloworld-695c77f7bd-wptgw
```

We can see that curl still works. Let's break it by setting a **STRICT** mTLS PeerAuthentication policy:

```bash
# kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT
EOF
```

Now let's see curl fail:

```bash
# kubectl run curltest -it --rm --image infoblox/dnstools
# curl http://helloworld.default.svc.cluster.local
curl: (56) Recv failure: Connection reset by peer
```

But just to make sure, external accesses via the Ingress Gateway should absolutely still work:

```bash
# curl --cacert temp/ca.crt https://helloworld.fireflyclass.com
Hello, world!
Version: 1.0.0
Hostname: helloworld-695c77f7bd-wptgw
```

So now, how do we make sure our debugging pod gets an automatic sidecar? Apparently, by labeling our Default namespace with `istio-injection=enabled`:

```bash
# kubectl label namespace default istio-injection=enabled
# kubectl run curltest -it --rm --image infoblox/dnstools
# curl http://helloworld.default.svc.cluster.local
Hello, world!
Version: 1.0.0
Hostname: helloworld-695c77f7bd-wptgw
```

Voilà! We have now enabled automatic mTLS in our cluster and automatic sidecar injection into pods so they are now part of the mesh.
