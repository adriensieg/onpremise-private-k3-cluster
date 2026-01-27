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
