![grafana with oauth](images/grafana-oauth.svg)

* [docker-ce 19.03.5](https://docs.docker.com/install/linux/docker-ce/fedora/)
* [k3d v1.7.0](https://github.com/rancher/k3d)
* [helm v3.1.2](https://helm.sh/)
* [metallb chart 0.8.1](https://github.com/metallb/metallb)
* [traefik 1.7.19](https://docs.traefik.io/v1.7/)
* [prometheus 0.37.0](https://github.com/coreos/prometheus-operator)
* [loki 1.4.1](https://github.com/grafana/loki)
* [fluent-bit 1.4.1](https://github.com/fluent/fluent-bit)
* [grafana 6.7.1](https://github.com/grafana/grafana)
* [external-auth-server 1.0](https://github.com/travisghansen/external-auth-server)
* [dex chart 2.9.0](https://github.com/helm/charts/tree/master/stable/dex)
* [glauth v1.1.2](https://github.com/glauth/glauth)

# Setup Grafana, Prometheus, Loki, and Fluent-bit in an on-prem Kubernetes
This is a guide on how I setup an LDAP-authenticated monitoring on my home Linux machine running k3d.

## TL;DR
1. Start glauth with a couple of users
2. Expose Dex https using metallb
3. Bind Dex to `ldap://glauth.auth-system.svc.cluster.local:389`
4. Use the oidc plugin in external-auth-server to authenticate against dex.auth-system.svc.cluster.local
5. Install Loki, Fluent-bit, Prometheus, and Grafana
6. Annotate Ingresses to forward auth to `http://external-auth-server.auth-system.svc.cluster.local`

We're going to:
* Expose Grafana, Prometheus, and Alertmanager using Traefik ingress
  - `https://monitoring.fireflyclass.com`
  - `https://prometheus.fireflyclass.com`
  - `https://alertmanager.fireflyclass.com`
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
	   --server-arg --no-deploy=servicelb                                                           \
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
      cookie: {
        domain: "fireflyclass.com",
      },
      custom_error_headers: {},
      custom_service_headers: {
        // add support for Grafana SSO
        "X-WebAuth-User": {
          // userinfo, id_token, access_token, refresh_token, static, config_token, plugin_config, req, parentRequestInfo
          source: "userinfo",
          query_engine: "jp",
          query: "$.name",
          query_opts: {
            // by default, a jsonpath query always returns a list (ie: array), this force the value to be the first value in the array
            single_value: true
          }
        },
        "X-WebAuth-Groups": {
          source: "userinfo",
          query_engine: "jp",
          query: "$.groups[*]"
        }
      },
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

Note `https://eas.fireflyclass.com` is exposed through our Traefik ingress controller, which simply takes the [direct redirect return from Dex](https://github.com/dexidp/dex/issues/448) (via a GET request parameter) to give us a `wildcard` redirect capability, which is super convenient if we want to use this oauth proxy for more than just the Grafana Dashboard. Of course, for this to work for our browser, we should add yet another entry to our `/etc/hosts` so our host can lookup the hostname since we're not running our own DNS server:

```bash
kubectl -n kube-system get services traefik

NAME   TYPE           CLUSTER-IP   EXTERNAL-IP    PORT(S)         AGE
traefik   LoadBalancer   10.43.200.211   172.18.100.0   80:32536/TCP,443:31335/TCP   127m
```

```
# /etc/hosts
172.18.100.0 eas.fireflyclass.com monitoring.fireflyclass.com prometheus.fireflyclass.com alertmanager.fireflyclass.com
```

## Install Loki and Fluent-bit
First get the loki git repo

```bash
git clone https://github.com/grafana/loki
```

Now install loki with a persistent storage to store 30 days worth of logs if possible.

```bash
kubectl create ns monitoring
helm upgrade --install --atomic --debug --force -n monitoring -f ./loki-values.yaml loki loki/production/helm/loki
```

```yaml
# loki-values.yaml
---
persistence:
  enabled: true
  size: 100Gi

config:
  table_manager:
    retention_deletes_enabled: true
    retention_period: 672h
```

We're going to use the Fluent-bit chart from the loki directory as it has hooks to forward to the loki service

```bash
helm upgrade --install --atomic --debug --force -n monitoring fluent-bit loki/production/helm/fluent-bit --set loki.serviceName=loki
```

## Install Prometheus and Grafana
Prometheus provides two webpages: `https://prometheus.fireflyclass.com` and `https://alertmanager.fireflyclass.com`. We're going to protect both using our oauth reverse proxy. We're not going to set any response headers since neither webpage cares about accounts. We will also put Grafana on `https://monitoring.fireflyclass.com` protected behind our oauth2 proxy, external-auth-server, set to auto-create accounts with the `Admin` role.

```bash
helm upgrade --install --atomic --debug --force -n monitoring -f ./prometheus-values.yaml prometheus-operator stable/prometheus-operator
```

```yaml
# prometheus-values.yaml
---
adminPassword: plaintext-password-for-"admin"-login

alertmanager:
  ingress:
    enabled: true
    annotations:
      ingress.kubernetes.io/auth-type: forward
      ingress.kubernetes.io/auth-url: "http://external-auth-server.auth-system.svc.cluster.local/verify?config_token_store_id=primary&config_token_id=1"
    hosts:
      - alertmanager.fireflyclass.com

prometheus:
  ingress:
    enabled: true
    annotations:
      ingress.kubernetes.io/auth-type: forward
      ingress.kubernetes.io/auth-url: "http://external-auth-server.auth-system.svc.cluster.local/verify?config_token_store_id=primary&config_token_id=1"
    hosts:
      - prometheus.fireflyclass.com

grafana:
  persistence:
    enabled: true
    accessModes:
      - ReadWriteOnce
    size: 10Gi

  grafana.ini:
    users:
      allow_sign_up: false
      auto_assign_org: true
      auto_assign_org_role: Admin
    auth.proxy:
      enabled: true
      header_name: X-WEBAUTH-USER
      header_property: username
      headers: Groups:X-WEBAUTH-GROUPS
      auto_sign_up: true

  ingress:
    enabled: true
    hosts:
      - monitoring.fireflyclass.com
    annotations:
      ingress.kubernetes.io/auth-type: forward
      ingress.kubernetes.io/auth-url: "http://external-auth-server.auth-system.svc.cluster.local/verify?config_token_store_id=primary&config_token_id=1"
      ingress.kubernetes.io/auth-response-headers: X-WebAuth-user, X-WebAuth-groups

  additionalDataSources:
    - name: loki
      type: loki
      access: proxy
      url: http://loki:3100
      editable: false
      jsonData:
        maxLines: 100000

  testFramework:
    enabled: false
```

You should already have `monitoring.fireflyclass.com` added to `/etc/hosts` as per the end of external-auth-server section, but here it is again to re-enforce that we are exposing `https://monitoring.fireflyclass.com` through our Traefik Ingress with our poorman's dns.

```bash
kubectl -n kube-system get services traefik

NAME   TYPE           CLUSTER-IP   EXTERNAL-IP    PORT(S)         AGE
traefik   LoadBalancer   10.43.200.211   172.18.100.0   80:32536/TCP,443:31335/TCP   127m
```

```
# /etc/hosts
172.18.100.0 eas.fireflyclass.com monitoring.fireflyclass.com prometheus.fireflyclass.com alertmanager.fireflyclass.com
```

Note, even though we are forwarding X-WEBAUTH-GROUPS but it is not used by Grafana (as of 6.7) so it's a noop really.

Note also that we are giving all users the default role of `Admin`. You should probably give them `Editor` or at least `Viewer`
