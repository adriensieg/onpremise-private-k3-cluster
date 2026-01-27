# Deployment of a multi-tenant Kubernetes Cluster

### The layout
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


```
# 1. Create secrets
kubectl apply -f auth/dex/secret.yaml
kubectl apply -f infrastructure/oauth2-proxy/secret.yaml

# 2. Deploy Dex
kubectl apply -f auth/dex/config.yaml
kubectl apply -f auth/dex/deployment.yaml
kubectl apply -f auth/dex/service.yaml
kubectl apply -f auth/dex/ingress.yaml

# 3. Deploy OAuth2 Proxy
kubectl apply -f infrastructure/oauth2-proxy/configmap.yaml
kubectl apply -f infrastructure/oauth2-proxy/deployment.yaml
kubectl apply -f infrastructure/oauth2-proxy/service.yaml

# 4. Update ingress
kubectl apply -f infrastructure/ingress/shared-ingress.yaml

# 5. Check status
kubectl get pods
kubectl logs -f deployment/dex
kubectl logs -f deployment/oauth2-proxy
```
