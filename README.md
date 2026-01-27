# Building a bare-metal Kubernetes cluster on Raspberry Pis

This architecture implements a **multi-tenant Kubernetes platform** where **each tenant** (**workspace**) operates as an **isolated namespace** with its **own subdomain**, **authentication configuration**, and **service deployments**.

<img width="600" height="800" alt="image" src="https://github.com/user-attachments/assets/4fb2e834-72f3-490d-91f5-fe5f9102b114" />

This project runs on a small self-hosted Kubernetes cluster:
- **Cluster type**: K3s
- **Hardware**: 2 × Raspberry Pi 4B
- **Nodes**:
    - **Control plane** (master): `10.0.0.50`
    - **Worker node**: `10.0.0.221`

- **Cloudflare Tunnel**: Provides a secure external endpoint and forwards all incoming traffic to a single internal service — the **NGINX Ingress Controller**. Cloudflare Tunnel (cloudflared): Acts as a secure, outbound-only connection from our cluster to Cloudflare, exposing internal services without opening firewall ports, often running as a DaemonSet or Deployment.

- **NGINX Ingress Controller**: Acts as the single entry point for HTTP(S) traffic, routing requests by path to the appropriate backend services. Manages external access, load balancing, and SSL termination within the cluster, routing traffic based on host/path. Requests can only reach applications through NGINX, and NGINX only injects identity headers after authentication.

- TLS/Certificate Management
    - Cloudflare handles TLS termination
    - Internal traffic is HTTP (within cluster)
    - Consider adding cert-manager for internal TLS if needed
  
- **Applications**: Exposed only internally via **ClusterIP services**;
  
- **Network isolation**: No external traffic reaches pods directly; all ingress goes through NGINX, and **ClusterIP services** are internal-only.

- **Internet Aware Proxy (IAP)**: Intercepts traffic for multiple protocols, including SSH, Kubernetes, HTTPS, and databases, and ensures that only authenticated clients can connect to target resources.

- **Domain**: `devlabai.work` purchased and managed on Cloudflare

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

- **OAuth2 Proxy**: Sits in front of our application, managing the OAuth/OIDC flow, redirecting users to an Identity Provider (IdP) for login, and setting authenticated user headers.
Identity Provider (IdP): Handles user authentication (e.g., Okta, Keycloak, GitHub).


- URL Mapping

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


- GitOps integration: Use ArgoCD or Flux for workspace deployment
- Resource quotas: Set limits per workspace namespace
- Network policies: Enforce inter-namespace communication rules
- Service mesh: Consider Istio/Linkerd for advanced traffic management
- Observability: Add Prometheus, Grafana, Jaeger
- CI/CD: Automate application deployment per workspace
- Multi-cluster: Extend architecture across multiple clusters
- External secrets: Integrate with Vault or cloud secret managers

Monitoring and Observability
Key Metrics to Track

- Pod health per namespace
- Ingress request rates per hostname
- Authentication success/failure rates
- Service latency
- Resource usage per workspace
