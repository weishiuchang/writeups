# `https://dashboard.fireflyclass.com`

1. `Browser` **>>>** `https://dashboard.fireflyclass.com`
    > `Browser` makes an HTTPS request to `Traefik` Controller

2. `Traefik` **>>>** `http://external-auth-server.auth-system.svc.cluster.local`
    > `Kubernetes Dashboard` Ingress asks `external-auth-server` to verify User
    > `external-auth-server` sees no cookie so it redirects `Browser` to `https://dex.auth-system.svc.cluster.local/`

3. `Browser` **>>>** `https://dex.auth-system.svc.cluster.local/`
    > `dexidp` webpage prompts User for login

4. `https://dex.auth-system.svc.cluster.local/` **>>>** `ldap://glauth:389`
    > `dexidp` checks given User login/password
    > also gets `groups` list from LDAP

5. `https://eas.fireflyclass.com` **<<<** `https://dex.auth-system.svc.cluster.local/`
    > `dexidp` creates JWT cookie and redirects `Browser` to `https://eas.fireflyclass.com`

6. `Traefik` **<<<** `https://eas.fireflyclass.com`
    > `external-auth-server` sets `Authorization: Bearer XXX` header
    > returns `200` to `Traefik`

7. `Browser` **<<<** `Traefik`
    > `Traefik` forwards `Browser` to `Kubernetes Dashboard` with `Bearer` token and `Domain Cookie`


# `https://grafana.fireflyclass.com`
1. `Browser` **>>>** `https://grafana.fireflyclass.com`
    > `Browser` makes an HTTPS request to `Traefik` Controller

2. `Traefik` **>>>** `http://external-auth-server.auth-system.svc.cluster.local`
    > `Grafana` Ingress asks `external-auth-server` to verify User
    > `external-auth-server` sees no cookie so it redirects `Browser` to `https://dex.auth-system.svc.cluster.local/`

3. `Browser` **>>>** `https://dex.auth-system.svc.cluster.local/`
    > `dexidp` webpage prompts User for login

4. `https://dex.auth-system.svc.cluster.local/` **>>>** `ldap://glauth:389`
    > `dexidp` checks given User login/password

5. `https://eas.fireflyclass.com` **<<<** `https://dex.auth-system.svc.cluster.local/`
    > `dexidp` creates JWT cookie and redirects `Browser` to `https://eas.fireflyclass.com`

6. `Traefik` **<<<** `https://eas.fireflyclass.com`
    > `external-auth-server` sets `X-WebAuth-User` header from `JWT` User name
    > returns `200` to `Traefik`

7. `Browser` **<<<** `Traefik`
    > `Traefik` forwards `X-WebAuth-User` to `Grafana` with `X-WebAuth-User` header
