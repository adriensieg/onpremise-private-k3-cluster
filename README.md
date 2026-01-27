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

- **NGINX Ingress Controller**: Acts as the single entry point for HTTP(S) traffic, routing requests by path to the appropriate backend services. Manages external access, load balancing, and SSL termination within the cluster, routing traffic based on host/path.

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


GitOps integration: Use ArgoCD or Flux for workspace deployment
Resource quotas: Set limits per workspace namespace
Network policies: Enforce inter-namespace communication rules
Service mesh: Consider Istio/Linkerd for advanced traffic management
Observability: Add Prometheus, Grafana, Jaeger
CI/CD: Automate application deployment per workspace
Multi-cluster: Extend architecture across multiple clusters
External secrets: Integrate with Vault or cloud secret managers


Monitoring and Observability
Key Metrics to Track

Pod health per namespace
Ingress request rates per hostname
Authentication success/failure rates
Service latency
Resource usage per workspace
