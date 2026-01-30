# Deployment of a multi-tenant Kubernetes Cluster

This guide covers the deployment of a **multi-tenant Kubernetes** architecture with 3 workspaces:
- **`public`**: Unauthenticated public landing page at devailab.work
- **`mcd`**: Authenticated workspace at mcd.devailab.work (Azure AD via Dex + OAuth2-Proxy)
- **`perso`**: Public workspace at perso.devailab.work (no authentication)

#### Authentication Architecture: 
- Only the **`mcd`** namespace includes **authentication** infrastructure
- **Authentication** components (**Dex**, **OAuth2-Proxy**) are **deployed per workspace**
- Each **authenticated workspace** can have its **own IDP configuration**

#### URL Routing
- Cloudflare tunnel routes all 3 domains to **NGINX Ingress Controller**
- **NGINX Ingress rules** route based on **hostname** and **path**
- **Each workspace** has its **own Ingress resource** in **its namespace**

# The deployment

### 1. Create Namespaces

```
kubectl apply -f infrastructure/namespaces.yaml
```

### 2. Setup Cloudflare Tunnel

1. Prepare Cloudflare Credentials - two files from Cloudflare:
    - `cert.pem` - Origin certificate
    - `credentials.json` - Tunnel credentials
 
2. Deploy Cloudflare Tunnel
```
kubectl apply -f infrastructure/cloudflare/secret.yaml
kubectl apply -f infrastructure/cloudflare/configmap.yaml
kubectl apply -f infrastructure/cloudflare/tunnel.yaml
```

3. Configure DNS in Cloudflare. Add CNAME records pointing to your tunnel:
    - `devailab.work` → <tunnel-id>.cfargotunnel.com
    - `mcd.devailab.work` → <tunnel-id>.cfargotunnel.com
    - `perso.devailab.work` → <tunnel-id>.cfargotunnel.com

<img width="75%" height="75%" alt="image" src="https://github.com/user-attachments/assets/94efcf2e-db7f-42dd-bbd9-2dfdf5dd3a5b" />

### 3. A. Deploy `Public` Workspace - without Authentication

[Deployment of a Service](https://github.com/adriensieg/onpremise-private-k3-cluster/edit/master/03-deployment-services.md)

```
<workspace>/
├── apps/
│   └── <app-name>/
│       └── k8s/
│           ├── <app>-deployment.yaml
│           └── <app>-service.yaml
└── ingress/
    └── ingress.yaml        # Routes hostname/* to services
```

- Deploy each service per Workspace (repeat for each service in the workspace)
```
# Deploy landing page
kubectl apply -f spaces/public/apps/landing/k8s/landing-deployment.yaml
kubectl apply -f spaces/public/apps/landing/k8s/landing-service.yaml
```

- Once all the services / applications have been deployed we can deploy the ingress

```
# Deploy ingress
kubectl apply -f spaces/public/ingress/ingress.yaml
```

- Verify:
```
kubectl get pods -n public
kubectl get svc -n public
kubectl get ingress -n public
```

### 3. B. Deploy `mcd` Workspace - with Authentication

```
<workspace>/
├── auth/                   # (Optional:) Only for authenticated workspaces
│   ├── dex-*               # Identity provider proxy
│   └── oauth2-proxy-*      # Authentication middleware
├── apps/
│   └── <app-name>/
│       └── k8s/
│           ├── <app>-deployment.yaml
│           └── <app>-service.yaml
└── ingress/
    └── ingress.yaml        # Routes hostname/* to services
```

##### Configure Azure AD Secrets

a. First, generate a cookie secret:
```
COOKIE_SECRET=$(openssl rand -base64 32) echo "Cookie Secret: $COOKIE_SECRET"
```

- Create a shared secret for Dex and OAuth2-Proxy:
```
OAUTH2_CLIENT_SECRET=$(openssl rand -base64 32) echo "OAuth2 Client Secret: $OAUTH2_CLIENT_SECRET"
```

- Update the secret files - `spaces/mcd/auth/dex-secret.yaml`
```
kubectl apply -f /tmp/dex-secrets.yaml
kubectl apply -f /tmp/oauth2-proxy-secrets.yaml
```

b. Configure Azure AD Application
In Azure Portal:
    - Register a new application
    - Add redirect URI: `https://mcd.devailab.work/dex/callback`
    - Create a **client secret**
    - Note down: **Client ID**, **Client Secret**, **Tenant ID**

c. Deploy Authentication Components
```
# Deploy Secrets
kubectl apply -f auth/dex/secret.yaml
kubectl apply -f infrastructure/oauth2-proxy/secret.yaml

# Deploy Dex
kubectl apply -f spaces/mcd/auth/dex-config.yaml
kubectl apply -f spaces/mcd/auth/dex-deployment.yaml
kubectl apply -f spaces/mcd/auth/dex-service.yaml
kubectl apply -f spaces/mcd/auth/dex-ingress.yaml

# Deploy OAuth2-Proxy
kubectl apply -f spaces/mcd/auth/oauth2-proxy-config.yaml
kubectl apply -f spaces/mcd/auth/oauth2-proxy-deployment.yaml
kubectl apply -f spaces/mcd/auth/oauth2-proxy-service.yaml
kubectl apply -f spaces/mcd/auth/oauth2-proxy-ingress.yaml

Verify:
kubectl get pods -n mcd | grep -E "dex|oauth2"
kubectl logs -n mcd -l app=dex
kubectl logs -n mcd -l app=oauth2-proxy
```

d. Deploy ALL of our Applications
```
# Deploy landing service
kubectl apply -f spaces/mcd/apps/landing/k8s/landing-deployment.yaml
kubectl apply -f spaces/mcd/apps/landing/k8s/landing-service.yaml

# Deploy helloworld service
kubectl apply -f spaces/mcd/apps/helloworld/k8s/helloworld-deployment.yaml
kubectl apply -f spaces/mcd/apps/helloworld/k8s/helloworld-service.yaml

# Deploy gemini realtime service
kubectl apply -f spaces/mcd/apps/geminirealtime/k8s/geminirealtime-deployment.yaml
kubectl apply -f spaces/mcd/apps/geminirealtime/k8s/geminirealtime-service.yaml

# Deploy main ingress
kubectl apply -f spaces/mcd/ingress/ingress.yaml

# Verify:
kubectl get pods -n mcd
kubectl get svc -n mcd
kubectl get ingress -n mcd
```

# The layout
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

