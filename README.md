# Building a bare-metal Kubernetes cluster on Raspberry Pis

<img width="50%" height="50%" alt="image" src="https://github.com/user-attachments/assets/0bf01d07-7f3f-4756-b86e-0e7bc7cca1aa" />

This project runs on a small self-hosted Kubernetes cluster:
- **Cluster type**: K3s
- **Hardware**: 2 × Raspberry Pi 4B
- **Nodes**:
    - **Control plane** (master): `10.0.0.50`
    - **Worker node**: `10.0.0.221`

- **Cloudflare Tunnel**: Provides a secure external endpoint and forwards all incoming traffic to a single internal service — the **NGINX Ingress Controller** at `http://ingress-nginx-controller.ingress-nginx.svc.cluster.local:80`. Cloudflare Tunnel (cloudflared): Acts as a secure, outbound-only connection from our cluster to Cloudflare, exposing internal services without opening firewall ports, often running as a DaemonSet or Deployment.

- **NGINX Ingress Controller**: Acts as the single entry point for HTTP(S) traffic, routing requests by path to the appropriate backend services. Manages external access, load balancing, and SSL termination within the cluster, routing traffic based on host/path. Requests can only reach applications through NGINX, and NGINX only injects identity headers after authentication.

- **TLS/Certificate Management**
    - Cloudflare handles **TLS termination**
    - **Internal traffic is HTTP** (within cluster)
    - ~~Consider adding cert-manager for internal TLS if needed~~
  
- **Applications**: Exposed only internally via **ClusterIP services**;
  
- **Network isolation**: No external traffic reaches pods directly; all ingress goes through NGINX, and **ClusterIP services** are internal-only.
  - Applications use **`ClusterIP`**
  - Internal DNS names only resolve inside the cluster
  - **Internet traffic cannot reach apps directly**

- **Internet Aware Proxy (IAP)**: Intercepts traffic for multiple protocols, including SSH, Kubernetes, HTTPS, and databases, and ensures that only authenticated clients can connect to target resources.

- **Domain**: `devlabai.work` purchased and managed on Cloudflare

This architecture implements a **multi-tenant Kubernetes platform** where **each tenant** (**workspace**) operates as an **isolated namespace** with its **own subdomain**, **authentication configuration**, and **service deployments**.
- **Namespace Isolation**. Each **workspace** operates in its own Kubernetes **namespace**:
    - `cloudflare` - Cloudflare tunnel
    - `public` - Public landing page
    - `mcd` - MCD workspace with authentication
    - `perso` - Personal workspace without authentication
    - **One namespace per workspace**: Each tenant gets a dedicated Kubernetes namespace.
    - **Resource isolation**: *Pods*, *services*, and *config* are **scoped to their namespace**.
    - **Independent lifecycles**: Each workspace can be *deployed*, *updated*, or *deleted* independently

- Keep **downstream apps** simple (**no auth logic**, **no JWT parsing**). Apps should be completely stateless regarding authentication. They just read “who is this user” and do their job.

- Maintain **support** for **multiple IdPs**

- Leverage **NGINX Ingress Controller** for **authentication enforcement**

- **URL Routing**
    - Cloudflare tunnel routes all 3 domains to **NGINX Ingress Controller**
    - **NGINX Ingress rules** route based on **hostname** and **path**
    - **Each workspace** has its **own Ingress resource** in **its namespace**

- **OAuth2 Proxy**: Sits in front of our application, managing the OAuth/OIDC flow, redirecting users to an Identity Provider (IdP) for login, and setting authenticated user headers. Auth gateway, validates sessions, injects user headers

- **Dex** is an **identity provider abstraction layer** that sits between **OAuth2 Proxy** and **Azure Entra ID**, translating Azure's authentication into standardized OIDC tokens. Value Brought:
    - **Protocol Translation**: Azure speaks Microsoft-specific OAuth2 → Dex converts to standard OIDC → OAuth2 Proxy only needs to understand one protocol
    - **Provider Independence**: Want to switch from Azure to Google, GitHub, LDAP, or SAML? Change Dex connector config only—OAuth2 Proxy configuration stays identical. Without Dex, you'd reconfigure OAuth2 Proxy for each provider's quirks.
    - **Multi-Provider Support**: Can enable multiple auth sources (Azure + Google + GitHub) simultaneously—Dex shows provider selection screen, OAuth2 Proxy sees unified OIDC interface.
    - **Token Normalization**: Different providers return claims in different formats/names—Dex standardizes claim structure before passing to OAuth2 Proxy.

- **Identity Provider (IdP)**: Handles user authentication (e.g., Okta, Keycloak, GitHub).

- **Authentication Boundaries**
    - **OAuth2-Proxy cookies** scoped to **workspace subdomain**
    - **No cookie sharing between workspaces**
    - **Each workspace can have different user access policies**


- **URL Mapping**:

| URL                                      | Namespace | Service           | Port | Auth |
|------------------------------------------|-----------|-------------------|------|------|
| https://devailab.work/                   | public    | landing           | 80   | No   |
| https://devailab.work/landing            | public    | landing           | 80   | No   |
| https://mcd.devailab.work/               | mcd       | landing           | 80   | Yes  |
| https://mcd.devailab.work/landing        | mcd       | landing           | 80   | Yes  |
| https://mcd.devailab.work/helloworld     | mcd       | helloworld        | 8000 | Yes  |
| https://mcd.devailab.work/gemini-r1      | mcd       | geminirealtime   | 8000 | Yes  |
| https://perso.devailab.work/             | perso     | landing           | 80   | No   |
| https://perso.devailab.work/landing      | perso     | landing           | 80   | No   |
| https://perso.devailab.work/gemini-r1    | perso     | geminirealtime   | 8000 | No   |

```
Internet
  │
  ▼
Cloudflare CDN (TLS termination)
  │
  ▼
Cloudflare Tunnel (encrypted connection to cluster)
  │
  ▼
cloudflared Pod (in cloudflare namespace)
  │
  ▼
NGINX Ingress Controller (in ingress-nginx namespace)
  │
  ├─ Host: devailab.work
  │  └─> Ingress in public namespace
  │      └─> Service in public namespace
  │          └─> Pod in public namespace
  │
  ├─ Host: mcd.devailab.work
  │  └─> Ingress in mcd namespace
  │      ├─> OAuth2-Proxy (auth check)
  │      │   └─> Dex (if needed)
  │      └─> Service in mcd namespace
  │          └─> Pod in mcd namespace
  │
  └─ Host: perso.devailab.work
     └─> Ingress in perso namespace
         └─> Service in perso namespace
             └─> Pod in perso namespace
```

# Layout of the project

```
infrastructure/
├── cloudflare/
│   ├── configmap.yaml          # Tunnel config for all 3 hostnames
│   ├── secret.yaml
│   └── tunnel.yaml
└── namespaces.yaml              # Namespaces: public, mcd, perso, cloudflare

spaces/
├── public/
│   ├── apps/
│   │   └── landing/
│   │       ├── app/
│   │       ├── Dockerfile
│   │       └── k8s/
│   │           ├── landing-deployment.yaml
│   │           └── landing-service.yaml
│   └── ingress/
│       └── ingress.yaml         # Routes devailab.work/ → landing
│
├── mcd/
│   ├── auth/
│   │   ├── dex-config.yaml
│   │   ├── dex-deployment.yaml
│   │   ├── dex-service.yaml
│   │   ├── dex-ingress.yaml
│   │   ├── dex-secret.yaml
│   │   ├── oauth2-proxy-config.yaml
│   │   ├── oauth2-proxy-deployment.yaml
│   │   ├── oauth2-proxy-service.yaml
│   │   ├── oauth2-proxy-ingress.yaml
│   │   └── oauth2-proxy-secret.yaml
│   ├── apps/
│   │   ├── landing/
│   │   │   ├── app/
│   │   │   ├── Dockerfile
│   │   │   └── k8s/
│   │   │       ├── landing-deployment.yaml
│   │   │       └── landing-service.yaml
│   │   ├── helloworld/
│   │   │   ├── app/
│   │   │   ├── Dockerfile
│   │   │   └── k8s/
│   │   │       ├── helloworld-deployment.yaml
│   │   │       └── helloworld-service.yaml
│   │   └── geminirealtime/
│   │       ├── app/
│   │       ├── Dockerfile
│   │       └── k8s/
│   │           ├── geminirealtime-deployment.yaml
│   │           └── geminirealtime-service.yaml
│   └── ingress/
│       └── ingress.yaml         # Routes mcd.devailab.work/* → services (auth required)
│
└── perso/
    ├── apps/
    │   ├── landing/
    │   │   ├── app/
    │   │   ├── Dockerfile
    │   │   └── k8s/
    │   │       ├── landing-deployment.yaml
    │   │       └── landing-service.yaml
    │   └── geminirealtime/
    │       ├── app/
    │       ├── Dockerfile
    │       └── k8s/
    │           ├── geminirealtime-deployment.yaml
    │           └── geminirealtime-service.yaml
    └── ingress/
        └── ingress.yaml         # Routes perso.devailab.work/* → services (public)
```

## Improvements

- **GitOps integration**: Use ArgoCD or Flux for workspace deployment
- **Resource quotas**: Set limits per workspace namespace
- **Network policies**: Enforce inter-namespace communication rules
- **Service mesh**: Consider Istio/Linkerd for advanced traffic management
- **Observability**: Add Prometheus, Grafana, Jaeger
- **CI/CD**: Automate application deployment per workspace
- **Multi-cluster**: Extend architecture across multiple clusters
- **External secrets**: Integrate with Vault or cloud secret managers

## Monitoring and Observability Key Metrics to Track
- **Pod health per namespace**
- **Ingress request rates per hostname**
- **Authentication success/failure rates**
- **Service latency**
- **Resource usage per workspace**
