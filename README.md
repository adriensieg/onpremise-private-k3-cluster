# Building a bare-metal Kubernetes cluster on Raspberry Pis

This architecture implements a **multi-tenant Kubernetes platform** where **each tenant** (**workspace**) operates as an **isolated namespace** with its **own subdomain**, **authentication configuration**, and **service deployments**.

<img width="600" height="800" alt="image" src="https://github.com/user-attachments/assets/4fb2e834-72f3-490d-91f5-fe5f9102b114" />

This project runs on a small self-hosted Kubernetes cluster:
- **Cluster type**: K3s
- **Hardware**: 2 × Raspberry Pi 4B
- **Nodes**:
    - **Control plane** (master): `10.0.0.50`
    - **Worker node**: `10.0.0.221`

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


```
| URL                                      | K8s Service       | Namespace | Auth Required      | Config File Path                              |
|------------------------------------------|-------------------|-----------|--------------------|-----------------------------------------------|
| https://devailab.work/landing             | landing           | public    | No                 | spaces/public/apps/landing/k8s/               |
| https://mcd.devailab.work/landing         | landing           | mcd       | Yes (Azure AD)     | spaces/mcd/apps/landing/k8s/                  |
| https://mcd.devailab.work/gemini-rl       | geminirealtime    | mcd       | Yes (Azure AD)     | spaces/mcd/apps/geminirealtime/k8s/           |
| https://mcd.devailab.work/helloworld      | helloworld        | mcd       | Yes (Azure AD)     | spaces/mcd/apps/helloworld/k8s/               |
| https://perso.devailab.work/landing       | landing           | perso     | No                 | spaces/perso/apps/landing/k8s/                |
| https://perso.devailab.work/gemini-rl     | geminirealtime    | perso     | No                 | spaces/perso/apps/geminirealtime/k8s/         |
```


Cloudflare Tunnel (cloudflared): Acts as a secure, outbound-only connection from your cluster to Cloudflare, exposing internal services without opening firewall ports, often running as a DaemonSet or Deployment.
NGINX Ingress Controller: Manages external access, load balancing, and SSL termination within the cluster, routing traffic based on host/path.
OAuth2 Proxy: Sits in front of your application, managing the OAuth/OIDC flow, redirecting users to an Identity Provider (IdP) for login, and setting authenticated user headers.
Identity Provider (IdP): Handles user authentication (e.g., Okta, Keycloak, GitHub). 


- ✅ **Domain**: devlabai.work purchased and managed on Cloudflare
- ✅ **Cloudflare Tunnel**: Secure encrypted tunnel from internet to your cluster
- ✅ **NGINX Ingress**: Single entry point routing traffic by path
- ✅ **Network Isolation**: Apps use ClusterIP (not exposed externally)
- ✅ **Our Apps**: Accessible at https://devlabai.work/xxx
- ✅ **High Availability**: 2 cloudflared replicas, NGINX ingress
- ✅ **No Port Forwarding**: No open ports on your home network
- ✅ **Free SSL**: Automatic HTTPS via Cloudflare
