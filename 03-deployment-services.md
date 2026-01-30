

## Workspace Layer
Each workspace follows a consistent structure:

```
<workspace>/
├── auth/                   # Optional: Only for authenticated workspaces
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

### Deploy `helloworld` service
```
# 1. Build Docker image. Ensure K3s can see the image (same Docker runtime).
docker buildx build --platform linux/arm64 -t helloworld:latest --load .

# 2. Transfer image to nodes
docker save helloworld:latest -o $env:TEMP\helloworld.tar

scp -i $HOME\.ssh\k3s-cluster $env:TEMP\helloworld.tar adsieg@10.0.0.50:/tmp/
scp -i $HOME\.ssh\k3s-cluster $env:TEMP\helloworld.tar adsieg@10.0.0.221:/tmp/

# 3. Then load it on both nodes:
ssh -i $HOME\.ssh\k3s-cluster adsieg@10.0.0.50 "sudo ctr -n k8s.io images import /tmp/helloworld.tar" docker.io/library/helloworld:latest
ssh -i $HOME\.ssh\k3s-cluster adsieg@10.0.0.221 "sudo ctr -n k8s.io images import /tmp/helloworld.tar" docker.io/library/helloworld:latest
### This ensures K3s nodes can run your ARM64 image locally without pulling from a registry.

# 4. Deploy app into apps namespace - we must be in 'C:\Users\cloti\Desktop\HandsOn\k3-homelab-prod>'
kubectl apply -n apps -f apps/helloworld/k8s/deployment.yaml
kubectl apply -n apps -f apps/helloworld/k8s/service.yaml
```

```
kubectl get pods -n apps

kubectl get pods -n apps -o wide
kubectl get svc -n apps
kubectl get ingress -n apps

# Use a temporary curl pod to test ClusterIP:
kubectl run tmp --rm -it --image=curlimages/curl --restart=Never -- curl -v http://helloworld.apps.svc.cluster.local:8000/helloworld

# Test NGINX Ingress
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- curl -H "Host: devailab.work" http://localhost/helloworld

# Ensure NGINX Ingress Controller is Running
# Check if NGINX ingress controller exists
kubectl get pods -n ingress-nginx

## Verify Cloudflare tunnel is running
kubectl get pods -n cloudflare
kubectl logs -n cloudflare -l app=cloudflared --tail=20

## Confirm apps namespace is clean
kubectl get all,ingress,configmap,secret -n apps

# Confirm cloudflare namespace still exists and is intact
kubectl get all,configmap,secret -n cloudflare
```
