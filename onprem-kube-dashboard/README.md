![dashboard with oauth](images/dashboard-oauth.svg)

* [docker-ce 19.03.5](https://docs.docker.com/install/linux/docker-ce/fedora/)
* [k3d v1.7.0](https://github.com/rancher/k3d)
* [helm v3.1.2](https://helm.sh/)
* [metallb chart 0.12.0](https://github.com/metallb/metallb)
* [traefik 1.81.0](https://docs.traefik.io/v1.7/)
* [kubernetes dashboard v2.0.0-rc7](https://github.com/kubernetes/dashboard)
* [external-auth-server](https://github.com/travisghansen/external-auth-server)
* [dex chart 2.9.0](https://github.com/helm/charts/tree/master/stable/dex)
* [glauth v1.1.2](https://github.com/glauth/glauth)

# Setup Kubernetes Dashboard in an on-prem Kubernetes
This is a guide on how I setup an LDAP-authenticated Kubernetes Dashboard on my home Linux machine running k3d.

## TL;DR
1. Start glauth with a couple of users and a couple of groups: admins and devops
2. Expose Dex https using metallb
3. Bind Dex to `ldap://glauth.auth-system.svc.cluster.local:389`
4. Use the oidc plugin in external-auth-server to authenticate against dex.auth-system.svc.cluster.local
5. Annotate Kubernetes Dashboard ingress to forward auth to `http://external-auth-server.auth-system.svc.cluster.local`
6. Bind cluster-admin roles to admins group and namespaced admins to devops group

We're going to:
* Expose Kubernetes Dashboard using Traefik ingress
  - `https://dashboard.fireflyclass.com`
* Expose Dex https using Metallb LoadBalancer (required)
  - `https://dex.auth-system.svc.cluster.local`
* Expose external-auth-server using Traefik ingress (wildcard redirect)
  - `https://eas.fireflyclass.com`
* Everything else remains internal to the Kubernetes cluster

## Prerequisites
* Linux
* Docker
* Helm (with stable repo added)
* kubectl

## Create a self-signed wildcard cert
kube-apiserver has a baked in requirement that it must use an http**s** in its `--oidc-issuer-url` argument. We must therefore setup Dex with a LoadBalancerIP in addition to a TLS certificate, so we'll use OpenSSL to generate a self-signed cert for dex.auth-system.svc.cluster.local that will also serve double duty in our Traefik wildcard domain `*.fireflyclass.com`. Using a single combined cert is not required but it does make this guide easier to document if we just worked with one certificate:

```bash
cat >/data/certs/san.conf<<EOF
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
basicConstraints = critical, CA:true
keyUsage = critical, keyCertSign, cRLSign, digitalSignature
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
# redundent - but required to make wildcard certs work
DNS.1   = *.fireflyclass.com
DNS.2   = fireflyclass.com
DNS.3   = dex.auth-system.svc.cluster.local
EOF

openssl req -newkey rsa:2048 -config /data/certs/san.conf -extensions v3_req -nodes -keyout /data/certs/fireflyclass.com.key -x509 -days 3650 -out /data/certs/fireflyclass.com.crt
```

Note the required `keyUsage = keyCertSign` as I found out that if you leave it off, then it is not a valid CA and will be [ignored by nodejs](https://github.com/travisghansen/external-auth-server/issues/70) in external-auth-server.

Also, note the *S*ubject *A*lternative *N*ame of `dex.auth-system.svc.cluster.local`: If I had a *real* DNS server then I would have used something that is addressable by services within Kubernetes and without, something like `dex.fireflyclass.com`.

## Start k3d
Before we can start k3d, we unfortunately need to edit our `/etc/resolv.conf` with a kludge that allow the created kube-apiservers to be able to resolve the Dex hostname `dex.auth-system.svc.cluster.local` for JWT. We do this kludge because k3d automagically mounts the host `/etc/resolv.conf` into the master-worker docker containers on start. You could of course, wait for the master-workers to start and then `docker exec` into each of them and edit `/etc/resolv.conf` individually, but this is easier:

```bash
vim /etc/resolv.conf

...
nameserver 10.43.0.10
...
```

```bash
k3d create --volume /data/certs:/oidc                                                                   \
           --publish 443:443                                                                            \
	   --no-deploy servicelb                                                                        \
           --server-arg --kube-apiserver-arg=oidc-issuer-url=https://dex.auth-system.svc.cluster.local/ \
           --server-arg --kube-apiserver-arg=oidc-client-id=oidc-auth-client                            \
           --server-arg --kube-apiserver-arg=oidc-groups-claim=groups                                   \
           --server-arg --kube-apiserver-arg=oidc-ca-file=/oidc/fireflyclass.com.crt                    \
           --workers 2
k3d get-kubeconfig
```

## Update Traefik with our wildcard cert
Update the TLS secret in our running Traefik with our fireflyclass.com.crt and fireflyclass.com.key. Make sure to use the base64 encoding for the secrets:

```bash
base64 -w0 /data/certs/fireflyclass.com.crt
base64 -w0 /data/certs/fireflyclass.com.key
kubectl edit -n kube-system secret traefik-default-cert
```

## Deploy glauth with users
Create an LDAP server with a couple of users we're going to use to test our Kubernetes Dashboard. Specifically, the accounts in the `admins` group will be given `cluster-admin` roles while accounts in the `devops` group will be given `admin` privileges to just the `devops` namespace in Kubernetes. We will also create a service account for Dex to bind (required).

Use the following commands to generate sha256 passwords (note the required `-n` to `echo`):
```bash
echo -n 'authbindpass' | openssl dgst -sha256
echo -n 'malpass' | openssl dgst -sha256
echo -n 'zoepass' | openssl dgst -sha256
echo -n 'kayleepass' | openssl dgst -sha256
```

|Login    |Password     |Groups   |
|---------|-------------|---------|
|authbind |authbindpass |svcaccts |
|mal      |malpass      |admins   |
|zoe      |zoepass      |devops   |
|kaylee   |kayleepass   |devops   |

```bash
kubectl create namespace auth-system
kubectl apply -n auth-system -f ./glauth.yaml
```

```yaml
# glauth.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: glauth-cfg
data:
  config.cfg: |
    debug = true
    [ldap]
      enabled = true
      listen = "0.0.0.0:389"
    [ldaps]
      enabled = false

    [backend]
      datastore = "config"
      baseDN = "dc=fireflyclass,dc=com"

    [[users]]
      name = "authbind"
      givenname = "dex"
      sn = "dex"
      mail = "dex@fireflyclass.com"
      unixid = 5001
      primarygroup = 5001
      passsha256 = "6bb2dbf3194b808a61489f3691a61c7438c7fc4a21313fb6c524c5b0a3262435"

    [[groups]]
      name = "svcaccts"
      unixid = 5001

    [[users]]
      name = "mal"
      givenname = "Mal"
      sn = "Reynalds"
      mail = "mal@fireflyclass.com"
      unixid = 5002
      primarygroup = 5002
      passsha256 = "b3a2a0b57be8aac56b24f83a36b0e69ca3af84c0ce42ab7ca5cdbfd2c5964a92"

    [[groups]]
      name = "admins"
      unixid = 5002

    [[users]]
      name = "zoe"
      givenname = "Zoe"
      sn = "Alleyne"
      mail = "zoe@fireflyclass.com"
      unixid = 5003
      primarygroup = 5003
      othergroups = [5004]
      passsha256 = "10e6ce702cb93e9ee7890ed7cc212c9fc2c78244f2a5939bb10c24015b701168"

    [[users]]
      name = "kaylee"
      givenname = "Kaylee"
      sn = "Frye"
      mail = "kaylee@fireflyclass.com"
      unixid = 5004
      primarygroup = 5003
      othergroups = [5005]
      passsha256 = "48db26463d21c097e219e91c26cb9f7798b00a4ac1309debb00a89e4747182e4"

    [[groups]]
      name = "devops"
      unixid = 5003

    # some othergroups for testing
    [[groups]]
      name = "washburnes"
      unixid = 5004

    [[groups]]
      name = "engineers"
      unixid = 5005

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: glauth
spec:
  selector:
    matchLabels:
      app: glauth
  template:
    metadata:
      labels:
        app: glauth
    spec:
      containers:
        - name: glauth
          image: glauth/glauth:v1.1.2
          ports:
            - containerPort: 389
          volumeMounts:
            - name: glauth-cfg
              mountPath: /app/config
      volumes:
        - name: glauth-cfg
          configMap:
            name: glauth-cfg

---
apiVersion: v1
kind: Service
metadata:
  name: glauth
spec:
  selector:
    app: glauth
  ports:
    - protocol: TCP
      port: 389
      targetPort: 389
```

## Deploy metallb with reachable IPs
k3d is just Kubernetes running in a Docker network on your host. We need to allocate IPs that are within the same `/16` subnet as the Kubernetes nodes for metallb to hand out LoadBalancerIPs that are reachable by both k8s pods and your browser (via docker0). Get the subnet to put into metallb with the `get nodes` command:
```bash
kubectl get nodes -o wide

NAME                       STATUS   ROLES    AGE   VERSION        INTERNAL-IP   EXTERNAL-IP   OS-IMAGE   KERNEL-VERSION           CONTAINER-RUNTIME
k3d-k3s-default-worker-0   Ready    <none>   43s   v1.17.3+k3s1   172.18.0.3    <none>        Unknown    5.4.17-200.fc31.x86_64   containerd://1.3.3-k3s1
k3d-k3s-default-worker-1   Ready    <none>   41s   v1.17.3+k3s1   172.18.0.4    <none>        Unknown    5.4.17-200.fc31.x86_64   containerd://1.3.3-k3s1
k3d-k3s-default-server     Ready    master   40s   v1.17.3+k3s1   172.18.0.2    <none>        Unknown    5.4.17-200.fc31.x86_64   containerd://1.3.3-k3s1
```

From the `INTERNAL-IP` column I decided to use `172.18.100.0/30`; As long as the range we pick is both in Docker's `/16` and high enough that it won't collide with what Docker will allocate then we should be fine.
```bash
helm upgrade --install --atomic --debug --force -f ./values.yaml -n kube-system metallb stable/metallb
```

```yaml
# values.yaml
configInline:
  address-pools:
    - name: static
      protocol: layer2
      addresses:
        - 172.18.100.0/30
```

## Deploy dex with our cert
```bash
kubectl create -n auth-system secret generic dex-web-server-tls --from-file=tls.crt=/data/certs/fireflyclass.com.crt --from-file=tls.key=/data/certs/fireflyclass.com.key
helm upgrade --install --atomic --debug --force -f ./values.yaml -n auth-system dex stable/dex
```

```yaml
# values.yaml
---
grpc: false
https: true

# we will be using our own cert in dex-web-server-tls
certs:
  web:
    create: false
  grpc:
    create: false

config:
  issuer: https://dex.auth-system.svc.cluster.local/
  staticClients:
  - id: oidc-auth-client
    redirectURIs:
    - 'https://eas.fireflyclass.com/oauth/callback'
    name: oidc-auth-client
    secret: oidc-auth-client
  connectors:
  - type: ldap
    id: ldap
    name: glauth
    config:
      host: glauth.auth-system.svc.cluster.local:389
      insecureNoSSL: true
      insecureSkipVerify: true
      bindDN: cn=authbind,ou=svcaccts,dc=fireflyclass,dc=com
      bindPW: authbind
      userSearch:
        baseDN: dc=fireflyclass,dc=com
        filter: "(objectClass=posixAccount)"
        username: uid
        idAttr: uid
        emailAttr: mail
        nameAttr: cn
      groupSearch:
        baseDN: ou=groups,dc=fireflyclass,dc=com
        filter: "(objectClass=posixGroup)"
        userAttr: uid
        groupAttr: memberUid
        nameAttr: uid

ports:
  web:
    servicePort: 443

service:
  type: LoadBalancer
```

Of note here is that the dex service is listening on 443 in the cluster, which Traefik 1.7 cannot handle as a backend which is why we are using Metallb to expose it externally to the cluster. *But*, this does mean that we need to be able to reach `dex.auth-system.svc.cluster.local` from our host/browser and from other pods in the cluster as the user browser will be redirected to it for a login prompt. This is solved by putting the LoadBalancedIP of the service into our host `/etc/hosts` file:

```bash
kubectl -n auth-system get services dex

NAME   TYPE           CLUSTER-IP   EXTERNAL-IP    PORT(S)         AGE
dex    LoadBalancer   10.43.8.80   172.18.100.1   443:32565/TCP   35s
```

```
# vim /etc/hosts
172.18.100.1 dex.auth-system.svc.cluster.local
```

Now verify the discovery url returns a good json: `curl -k https://dex.auth-system.svc.cluster.local/.well-known/openid-configuration`

## Deploy external-auth-server
Let's clone the `external-auth-server` repository as we will need to generate a configuration token using the provided nodejs script:
```bash
git clone https://github.com/travisghansen/external-auth-server.git
vim external-auth-server/bin/generate-config-token.js
```

```javascript
// ./external-auth-server/bin/generate-config-token.js
eas: {
  plugins: [
    // copy and paste the oidc section from external-auth-server/PLUGINS.md
    {
      type: "oidc",
      issuer: {
        discover_url: "https://dex.auth-system.svc.cluster.local/.well-known/openid-configuration",
      },
      client: {
	client_id: "oidc-auth-client",
	client_secret: "oidc-auth-client",
      },
      scopes: ["openid", "email", "profile", "groups"],
      custom_authorization_parameters: {},
      redirect_uri: "https://eas.fireflyclass.com/oauth/callback",
      features: {
	cookie_expiry: false,
	userinfo_expiry: true,
        session_expiry: true,
        session_expiry_refresh_window: 86400,
        session_retain_id: true,
        refresh_access_token: true,
        fetch_userinfo: true,
        introspect_access_token: false,
        authorization_token: "access_token",
      },
      assertions: {
        exp: true,
        nbf: true,
        iss: true,
      },
      xhr: {},
      cookie: {},
      custom_error_headers: {},
      custom_service_headers: {},
    },
  ],
```

Now generate a couple of random characters for `EAS_CONFIG_TOKEN_SIGN_SECRET` and `EAS_CONFIG_TOKEN_ENCRYPT_SECRET` and run external-auth-server/bin/generate-config-token.js to get the encrypted output. Make sure to save those random characters for values.yaml as they are needed for decrypting:
```bash
EAS_CONFIG_TOKEN_SIGN_SECRET=`openssl rand -base64 6`
EAS_CONFIG_TOKEN_ENCRYPT_SECRET=`openssl rand -base64 6`
docker run -it --rm -e EAS_CONFIG_TOKEN_SIGN_SECRET=$EAS_CONFIG_TOKEN_SIGN_SECRET -e EAS_CONFIG_TOKEN_ENCRYPT_SECRET=$EAS_CONFIG_TOKEN_ENCRYPT_SECRET -v $PWD/external-auth-server/bin/generate-config-token.js:/home/eas/app/bin/generate-config-token.js travisghansen/external-auth-server node bin/generate-config-token.js
```

This should print out two long strings. We only need the first one - `encrypted token` for server-side usage. Copy and paste the entire token and place it in values.yaml as a `configTokens`:
```bash
# helm upgrade --install --atomic --debug --force -f ./values.yaml -n auth-system external-auth-server external-auth-server/charts/external-auth-server/
```

```yaml
# values.yaml
---
logLevel: "debug"

# each of these secrets must be unique and distinct
configTokenSignSecret: <this from $EAS_CONFIG_TOKEN_SIGN_SECRET above>
configTokenEncryptSecret: <this is from $EAS_CONFIG_TOKEN_ENCRYPT_SECRET above>

# create some random characters from `openssl rand -base64 6`
issuerSignSecret: 7eN8RyNr
issuerEncryptSecret: J8OXg8i2
cookieSignSecret: lnIGI2il
cookieEncryptSecret: U7j6TQ3I
sessionEncryptSecret: KOIpmznE
storeOpts: {}
revokedJtis: []

configTokenStores:
  primary:
    adapter: env
    options:
      cache_ttl: 3600
      var: EAS_CONFIG_TOKENS

# copy outputted server-side encrypted token created by bin/generate-config-token.js
configTokens:
  1: rzL49fQK8Dm9w8JO98qM1QLJ7aApDK0p1CmJ1LwQ+QWgEg0dZjF8smeHtbvtk/PMQrUSraqWXNq9/k1rEzpjUhVhBQQrQA/bv0+OF/9wlInziSbAZ8X6s2w2qpCKnsY97GFkGNOosi9AkjIS3wakKD0XxhYAUTbTjlXRq4ycqlwXR6xcMRyM3pYR0BEyCHDRlq3jvor5AFL9wLK0/noU71l5+/I61AYUSbxz0ZMRNZHjlhD0xfea01Dr6yPDUBitFbQfq3u2WfebrtS4BmpHlNnucRo60YTSR7KCj23uk5t2EG15xpk5OqHh99KanWjs0DXmusv3BjcgyQq/vIK3vIaxR5e6L1OBLKfDyUBsTBuVZm9bub3BbYo/x74rUh0L5G9HvDcXLUnlfNK2x2UvG1BLP2Q7H+XrFqElj/zE5EQ1CMFHnHEZDT84FQaFq2RitphlVNZ9k5SFYu2+5xj0QhqcE3+2uCb97r/GMz0zgwngu0Oc/wbF/26w0siiOVefQSgZd7wbtDJJiWHYPo4LTZmNndLgHvQjjdJezFI5777YXDZ30loi/qN2b0FPEmfCGZaAhXc4PlRUnpkNGZegaDJ2ggEBjqvIbXA+ecpveBY57qx06jDXLZeiLv6T1sBrneOFeX4jN9RzezRvSMiri6EbnIXT079ypi+9RdRmaVhu3BUIkAoB2HW7vgwu6BLTCh3ivebRD8xixayqHBN/JrQrNc68CZklOmTFFlsaDaZ1Z2WayDHQtD+LVmrl+LFc0o+Lr4oQl/Pb4PbIHfMyZrWmCVr6KAmlkVpes00zNHgh8ImYKs3zwuj4HjPyVRkjZlhwF0zbkHcNyFl72I1HGkHie6LrHF9AQCegt3bz4AMO3wyieG6E1ENFHFq9bMM11cJ+6k9VUerWSW9Ft7+1ofH/Nzfu7WpI1GnlcNNURRk684rGP+pxip+aMJhppdmVE7F/GllzvG88ShuEOIJ75br4jG5yS7Mp+hsBXK+FRxGwQUUwjI6A8JJ2PDj7jRnesT4PMQ/TCNJl9yN/ldcF+xOb0j6bSUWXfsbQEMqigWYMT3kanMuAH2Bw4iHEJrCUDzFBOtok+LQjQwVvf1QEoGI64gLFtQ4ZlRKiLHZCMoYnRuU1NIRvKi4Cj3C/TZR8OZmn2Q2ni18g1jGtDWLM07iYf0xTiCbWty+LETP1kRTDvrS2qGGEkTBJ6X7PDcCI93TToFiMe9UnYD1FyAILgEEQ3Idz8cahAX5cdVLDd+6MGltav4T8W9bRvXKE6cNxPQTPHVSu7xXjPRuyxDmOqSEoe2UExhR3QYnnNFH3f7mVCXnD5o1Gkmn53D5Sc9husEAKa/A6oAHLcXxn7qUDvnI1L3o+GyNDRKhJTvoIMTx1p61fPO8TU9GgoT/by757a55LkakjZDsGZIarxSzqH8EUnXDHWzT58eePUR7sfao/fpUzyu/WPk6bHDni/tRTk2F43qlUVLRgixYwDGHxtg==

# this should be the plain contents of /data/certs/fireflyclass.com.crt
nodeExtraCaCerts: |-
  -----BEGIN CERTIFICATE-----
  MIIEBTCCAu2gAwIBAgIJAOMKQ9GM2CvZMA0GCSqGSIb3DQEBCwUAMF4xCzAJBgNV
  ...
  -----END CERTIFICATE-----
  
securityContext:
  enabled: true

# required since dex does not allow wildcard redirect
ingress:
  enabled: true
  paths:
    - /
  hosts:
    - eas.fireflyclass.com
```

Note `https://eas.fireflyclass.com` is exposed through our Traefik ingress controller, which simply takes the [direct redirect return from Dex](https://github.com/dexidp/dex/issues/448) (via a GET request parameter) to give us a `wildcard` redirect capability, which is super convenient if we want to use this oauth proxy for more than just the Kubernetes Dashboard. Of course, for this to work for our browser, we should add yet another entry to our `/etc/hosts` so our host can lookup the hostname since we're not running our own DNS server:

```bash
kubectl -n kube-system get services traefik

NAME   TYPE           CLUSTER-IP   EXTERNAL-IP    PORT(S)         AGE
traefik   LoadBalancer   10.43.200.211   172.18.100.0   80:32536/TCP,443:31335/TCP   127m
```

```
# /etc/hosts
172.18.100.0 eas.fireflyclass.com dashboard.fireflyclass.com
```

## Deploy Kubernetes Dashboard with auth forward
We're going to deploy the dashboard as plain http (port 80) in the cluster and use the Traefik Ingress Controller to terminate http**s** for us, and we will also use annotations to instruct Traefik 1.7 to forward authentication to our internal `http://external-auth-server.auth-system.svc.cluster.local` oauth proxy:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc7/aio/deploy/alternative.yaml
kubectl apply -f ./dashboard-ingress.yaml
```

```yaml
# dashboard-ingress.yaml
---
apiVersion: extensions/v1beta1
kind: Ingress

metadata:
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
  annotations:
    ingress.kubernetes.io/auth-type: forward
    ingress.kubernetes.io/auth-url: "http://external-auth-server.auth-system.svc.cluster.local/verify?config_token_store_id=primary&config_token_id=1"
    ingress.kubernetes.io/auth-response-headers: X-Userinfo, X-Id-Token, X-Access-Token, Authorization

spec:
  rules:
    - host: dashboard.fireflyclass.com
      http:
        paths:
          - backend:
              serviceName: kubernetes-dashboard
              servicePort: 80
            path: /
```

You should already have `dashboard.fireflyclass.com` added to `/etc/hosts` as per the end of external-auth-server section, but here it is again to re-enforce that we are exposing `https://dashboard.fireflyclass.com` through our Traefik Ingress.

```bash
kubectl -n kube-system get services traefik

NAME   TYPE           CLUSTER-IP   EXTERNAL-IP    PORT(S)         AGE
traefik   LoadBalancer   10.43.200.211   172.18.100.0   80:32536/TCP,443:31335/TCP   127m
```

```
# /etc/hosts
172.18.100.0 eas.fireflyclass.com dashboard.fireflyclass.com
```

## Add Role Bindings
And finally we grant permissions to our intrepid users. First Lets give all users in the `admins` group ability to perform all any action on any resource in the entire cluster:

```bash
kubectl apply -f ./admins-group.yaml
```

```yaml
# admins-group.yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admins-group
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: Group
  name: admins
```

Now create a `devops` namespace and grant users in the `devops` group full permission to that namespace:

```bash
kubectl apply -f ./devops-group.yaml
```

```yaml
# devops-group.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: devops

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devops-group
  namespace: devops
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin
subjects:
- kind: Group
  name: devops
```

# Bonus configuration
With glauth and Dex in place, we can deploy [dex-k8s-authenticator](https://github.com/mintel/dex-k8s-authenticator) to provide `kubeconfig` oauth tokens to our users. This creates an ingress for `https://kubectl.fireflyclass.com` for our users.

As we won't be using external-auth-server proxy for dex-k8s-authenticator, this will require a small change to Dex's deployment: The addition of a redirect url:
```yaml
# dex's values.yaml
---
...
config:
  ...
  staticClients:
  - id: oidc-auth-client
    redirectURIs:
    - 'https://eas.fireflyclass.com/oauth/callback'
    - 'https://kubectl.fireflyclass.com/callback/fireflyclass'
    name: oidc-auth-client
    secret: oidc-auth-client
...
```

```bash
git clone https://github.com/mintel/dex-k8s-authenticator.git
helm upgrade --install --atomic --debug --force -f ./values.yaml -n auth-system dex-k8s-authenticator dex-k8s-authenticator/charts/dex-k8s-authenticator/
```

```yaml
# values.yaml
---
dexK8sAuthenticator:
  debug: true
  tlsCert: /certs/dex-client.crt
  tlsKey: /certs/dex-client.key
  clusters:
  - name: fireflyclass
    short_description: "Serenity"
    description: "Serenity Cluster"
    client_id: oidc-auth-client
    client_secret: oidc-auth-client
    issuer: https://dex.auth-system.svc.cluster.local/
    redirect_uri: https://kubectl.fireflyclass.com/callback/fireflyclass
    # k3d deployment has kube-apiserver listening on :6443
    k8s_master_uri: https://127.0.0.1:6443

caCerts:
  enabled: true
  secrets:
  - name: kube-apiserver-cert
    filename: kube-apiserver.crt
    # echo | openssl s_client -connect localhost:6443 2>/dev/null | openssl x509 | base64 -w0
    value: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUJ6RENDQVhLZ0F3SUJBZ0lJTWExZzdiS1JZMGd3Q2dZSUtvWkl6ajBFQXdJd0l6RWhNQjhHQTFVRUF3d1kKYXpOekxYTmxjblpsY2kxallVQXhOVGcyTURJeE5USTNNQjRYRFRJd01EUXdOREUzTXpJd04xb1hEVEl4TURRdwpOREUzTXpJd09Gb3dIREVNTUFvR0ExVUVDaE1EYXpOek1Rd3dDZ1lEVlFRREV3TnJNM013V1RBVEJnY3Foa2pPClBRSUJCZ2dxaGtqT1BRTUJCd05DQUFUK1ZONjJ3Y1FsVjBXVm5HWEk5TVhHZ0c3cjUyeVpxVjBtTVBZNlpaOEsKS041NmdQS3pHZVZ4K2xIVFBlL2tkREtTbWpzYW1wYjRxNTdtMGdza3IwK0JvNEdXTUlHVE1BNEdBMVVkRHdFQgovd1FFQXdJRm9EQVRCZ05WSFNVRUREQUtCZ2dyQmdFRkJRY0RBVEJzQmdOVkhSRUVaVEJqZ2dwcmRXSmxjbTVsCmRHVnpnaEpyZFdKbGNtNWxkR1Z6TG1SbFptRjFiSFNDSkd0MVltVnlibVYwWlhNdVpHVm1ZWFZzZEM1emRtTXUKWTJ4MWMzUmxjaTVzYjJOaGJJSUpiRzlqWVd4b2IzTjBod1FLS3dBQmh3Ui9BQUFCaHdTc0VnQUNNQW9HQ0NxRwpTTTQ5QkFNQ0EwZ0FNRVVDSUZZZFN1MDNwb1crTEtMOXBRSG1lRWhGUGtYZ1JXUUZ2dlRLVHo0Rk0wcUlBaUVBCnF1VFV3MmhjWWU0V3RLbUE1SE16SjBBVldKL01EbXI3dHRJYnJUOFhacEk9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
  - name: dex-cert
    filename: dex-client.crt
    # base64 -w0 /data/certs/fireflyclass.com.crt
    value: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUVJRENDQXdpZ0F3SUJBZ0lVSVpmL1ZCVnp4MHVsNW1kWFo1Y0htRXJYT1VNd0RRWUpLb1pJaHZjTkFRRUwKQlFBd1lqRUxNQWtHQTFVRUJoTUNWVk14RVRBUEJnTlZCQWdNQ0VOdmJHOXlZV1J2TVJBd0RnWURWUVFIREFkQwpiM1ZzWkdWeU1SRXdEd1lEVlFRS0RBaEVjbTlzY21WMmJ6RWJNQmtHQTFVRUF3d1NLaTVtYVhKbFpteDVZMnhoCmMzTXVZMjl0TUI0WERUSXdNRFF3TkRJd016VXlObG9YRFRNd01EUXdNakl3TXpVeU5sb3dZakVMTUFrR0ExVUUKQmhNQ1ZWTXhFVEFQQmdOVkJBZ01DRU52Ykc5eVlXUnZNUkF3RGdZRFZRUUhEQWRDYjNWc1pHVnlNUkV3RHdZRApWUVFLREFoRWNtOXNjbVYyYnpFYk1Ca0dBMVVFQXd3U0tpNW1hWEpsWm14NVkyeGhjM011WTI5dE1JSUJJakFOCkJna3Foa2lHOXcwQkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXVVM0ZUYmlHMUV6dlhjWFNnWHFrakh0RzVyOE8KZDhWbGh3ZEtGOTZ1VGVJM2ovUktHbEh4Wnp6OVY5ajV2Y2oxclVmVGFKeDNXYkFzWVowZkEyNXNobDNvS2sxNQppbkRscjBSZzIyWWkyUTVHZXZscGpieWhkRFdtQ3FSSHAwZFM5SFdGQWZvNHIwT3FUdTZTY0FIUitYSzVaUUlUCnU3dHVFTWRQVjcvVDVROVpBMXVKSFRXSWM5MGZZZ1dLVmVLbnByQ0RYb1MrdTlORGpBQ2IxUllKTWlHYzB1aGoKamMwSzBaUmlMeW9Ja0JQQ0VRMmFwMFZEWk9VMElaWTJLdTRTR3RKdkVoSktIN1ZJVDhGMk5ERTFsZXh4T0UxSgpVSW96aDZ3QkRvRUdwK3g2bExJbW0ydnQrNVZxMHVGWVlaSmozWlJrM0x0c1RqeWJNUVBZR3pmbmh3SURBUUFCCm80SE5NSUhLTUE4R0ExVWRFd0VCL3dRRk1BTUJBZjh3RGdZRFZSMFBBUUgvQkFRREFnR0dNQjBHQTFVZERnUVcKQkJSaTV2bFZpY21VeXpPek5DUmdNTUFZUkZsY0lqQWZCZ05WSFNNRUdEQVdnQlJpNXZsVmljbVV5ek96TkNSZwpNTUFZUkZsY0lqQVRCZ05WSFNVRUREQUtCZ2dyQmdFRkJRY0RBVEJTQmdOVkhSRUVTekJKZ2hJcUxtWnBjbVZtCmJIbGpiR0Z6Y3k1amIyMkNFR1pwY21WbWJIbGpiR0Z6Y3k1amIyMkNJV1JsZUM1aGRYUm9MWE41YzNSbGJTNXoKZG1NdVkyeDFjM1JsY2k1c2IyTmhiREFOQmdrcWhraUc5dzBCQVFzRkFBT0NBUUVBR1V3bXJTVmZndWlEUjJORApiV3FHS0JkelpQaGFUaVNuZ1JCTGVodHl5dGdNdm9Canl4T2cwMmlFWDlwMGhWNzVUc2ErMEc2V1E4eFVvRU0rCmxaQzF0b2xFbHdmdnRlb0s5emZGenZnUkRjdDJOak92SUh4blVBM2Nxc3lGNG1MdUlXK0JhM2VRSHNmN0h0WDQKVkh1cm41UFdVaUFnTEo1RHg3d3lUSFpDYlg2blJ1VVpoWlVDK01Fb1BpcEVrbUgxWWo5SHkwc0ZqRk9sOGFLcQo2bkg1L3VnallraVVsQTNkbGRRRVliZ3FTdHJhdnUyS2xXWkFLdDE1MHpqSU45bk1lOFBPZVo2ZzJnYTVJQjFnCnB2d3FWTlRnS2hEeTlFSzN4SGRPeEY3VWIxM01HMElNZWhmMmI2aHlJcHFqUWcvUnNEd1BjeGZNN0RHNlJLZWQKNXRYanpnPT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
  - name: dex-key
    filename: dex-client.key
    # base64 -w0 /data/certs/fireflyclass.com.key
    value: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUV2UUlCQURBTkJna3Foa2lHOXcwQkFRRUZBQVNDQktjd2dnU2pBZ0VBQW9JQkFRQzVUY1ZOdUliVVRPOWQKeGRLQmVxU01lMGJtdnc1M3hXV0hCMG9YM3E1TjRqZVA5RW9hVWZGblBQMVgyUG05eVBXdFI5Tm9uSGRac0N4aApuUjhEYm15R1hlZ3FUWG1LY09XdlJHRGJaaUxaRGtaNitXbU52S0YwTmFZS3BFZW5SMUwwZFlVQitqaXZRNnBPCjdwSndBZEg1Y3JsbEFoTzd1MjRReDA5WHY5UGxEMWtEVzRrZE5ZaHozUjlpQllwVjRxZW1zSU5laEw2NzAwT00KQUp2VkZna3lJWnpTNkdPTnpRclJsR0l2S2dpUUU4SVJEWnFuUlVOazVUUWhsallxN2hJYTBtOFNFa29mdFVoUAp3WFkwTVRXVjdIRTRUVWxRaWpPSHJBRU9nUWFuN0hxVXNpYWJhKzM3bFdyUzRWaGhrbVBkbEdUY3UyeE9QSnN4CkE5Z2JOK2VIQWdNQkFBRUNnZ0VBYnIvRmVYTWMya3YxRUFXSWo2dytKbHdUZnh1WFNRd29adkI3WHhMTytHdUkKcTdma2hpQ05lQmRpanh0MS8vRFlyS3p0OUdyM2hob2VIR1Vzd1A5QlMzbGFwZFhTRzJUb1ViMDdha1Y3OUdCcwp6Vmk5dG1HVDJZR3E4RmRKSC9nbjQwVk5ybVhmZFJpcTlDdndSNU0rN0tpZGwzb0xVenR0U0Fmbkt0blNpZWFDCjJGeDZXV1lMbjNXZklJbnVsUGxhZitnclFwdkNzckhnTlgrYS9yMHJjVzhpUDlQM3hQMVV1bHhyNDFXVUVtVzkKWHdMdVY1ZnU5MFRpeThNMC9qamxEem5yN2doQnRiSTcvbkY2N3ZkU0FmSG00S0JWU2lxdXR0TzF1RmFpcW5hKwpweDh3U3ducDNHZ2pPWGlYNXpLYTZ6ZjE2VEZnZkJ1M3c2Ry9BaHM3aVFLQmdRRGh0RFFrMkNEbHN0ZzVDa21hCkU3RDlxQnFsbE1DZUthbWx6RjZyZ0pzTXkxc0pBZGJRNkFTMFZvdFR2czU2Qk1oQ0laU2NLL1BOcWpVdWFwT1EKTTlFNkIwaGZSbU5CeUlzUW4zL2o1blpzaStpY1FRUG5XMENVb0RqOXZZZ2cxZzZzU05sUUJ5TmUvSVc4OEVnQQpiZUFHNlNOYm1yTy8wQjZKcGhwM2NpV2VDd0tCZ1FEU0xVODZxZk9HZlc1WUg0R2xKTGwrZSt5YnI4eWcxdFk0Clk2U0dFMWYvckFTOXhTVmRwak5BZ0xiSVNSRnBpZWR1a2syWUc3eTFXT01iQmhtTklHTUpWNTI2RXBpQlk5YVIKYnZRODJHclFUSE9uV3hmYTVKb1hERXVVT2pGWE9MS2QzRWhEbG9seitBSlhsUzFsYXNGM2JyVUtSUnpzZVlxRQpmNGs1dS9wVjlRS0JnRmlCVkkwNkh6UlRmRXhweDFEZTllR1I0TmtiU3FqNngyYVhqR3dPSXo3U0kyR1YwZ25iClliVGgxd0xBNkxDYVhYanBPQ0JCYi9vdkMybW5LelE1ellyR3ZrOTJCNGdOUHRNRzZKeVNpOCttMFZFc2dYNWcKbnlObzdOQTdXVDBmRTJQbHNTbWJrdmcxWjdBZVBPM0dLRG90ZzhyeEVCbGdZQWswRkY3UWFRWGZBb0dBQ2xsVwp6R0d2N3hCZ0RaREhsblVmZVIzckFhTi9aUEFQTGttaHdVUlVrZTlMY0hpenBVL1l1RFZlU3JCbVhoYi9RVStNCjZJOTlGRVRqTWVKMEFhSDFubkVsQUJPSVZONndvd3FlbGh4bDdnbkZyQmp0TE1jUzIwMnNyd2pLZ3l2MGg1ZGcKSzR2UEN0bk1hN21adWFPdkVRMXZkcWoraXBwVmVyNjQ2QkhjZXIwQ2dZRUFwV3VtTWpsMDIxb2Y1TkVuS09NbwpDTkVoaUZLb1ZqMXB6bXhGbFJNanFIWGJ2ZVk3SEFaRE1XMGtsdllVUHFqcE1KMHN0Z3ZyVTFQMWdOSnNnek0xCnF1aXJETW52VW9OOUxRNEhISlpwN0tRblNaQkF3MVV2WFVmcjlBRVRsU0Zmc2tERzd6Mk9tWDl3ZGc3Zk80SDgKMURXOXZxRTNTd0xWQkt6UFhtQkFnZW89Ci0tLS0tRU5EIFBSSVZBVEUgS0VZLS0tLS0K

ingress:
  enabled: true
  path: /
  hosts:
    - kubectl.fireflyclass.com
```
