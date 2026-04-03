
# Situation

- **GitHub Actions** builds & pushes an **ARM64 Docker image** to **GHCR**
- *ArgoCD* detects the new image tag and syncs your **K8s manifests**. ArgoCD runs inside our cluster and pulls from GitHub. It never needs inbound internet access

- **K3s** deploys the new pods
- **NGINX** + **Cloudflare** serve it publicly

- arm64 or amd64
- arm64 or x64

## Implementation
- **Install ArgoCD on our K3s cluster**: SSH into our control plane (we need to be on your local network for this initial setup)
- **Restructure our GitHub repository**: Our repo needs a clear separation between app source code and Kubernetes manifests. ArgoCD watches the manifests; GitHub Actions updates them.
- **Configure GHCR access**: Our cluster needs credentials to pull images from GHCR. Create a GitHub Personal Access Token (PAT) with read:packages scope at `https://github.com/settings/tokens`. Then create the pull secret in each namespace that needs it. 
- **GitHub Actions workflow**: Create `.github/workflows/deploy.yaml`. This workflow detects which app changed and only rebuilds that app.
- **Configure ArgoCD Applications**: Create one ArgoCD Application resource per workspace. ArgoCD will watch that folder in our repo and keep the cluster in sync
- **Repository secrets**: That's it. GITHUB_TOKEN is automatically injected by Actions and has write access to GHCR for your repo. No additional secrets needed for the basic flow.
- The new deployment workflow
- Monitoring deployments


ArgoCD watches your Git repo for manifest changes. When GitHub Actions updates the image tag in deployment.yaml and pushes that commit, ArgoCD detects the diff and automatically syncs the new deployment to your cluster. This is the GitOps pattern — Git is the single source of truth.

You can push from anywhere — your laptop at a café, your office, anywhere with internet. ArgoCD on your cluster polls GitHub every 3 minutes (or you can configure a webhook for instant sync).

### 1. Install ArgoCD on our K3s cluster. 

SSH into our **control plane** (we need to be on **our local network** for this initial setup)

```
# Create the ArgoCD namespace and install
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all pods to be ready
kubectl wait --for=condition=Ready pod --all -n argocd --timeout=300s

# Install the ArgoCD CLI on your local machine (Windows)
winget install ArgoProj.ArgoCD
```

# Architecture

<img width="75%" height="75%" alt="image" src="https://github.com/user-attachments/assets/b870d09e-b42a-43ab-ae68-795f98d6a8f6" />


<img width="800" height="370" alt="image" src="https://github.com/user-attachments/assets/fb58951a-e74f-4a30-b4d8-22633305aa7c" />


<img width="1646" height="986" alt="image" src="https://github.com/user-attachments/assets/09d46404-adcd-487e-89af-d0b2691cbb54" />


# Bibliography

- [Deploy AI agents on Amazon Bedrock AgentCore using GitHub Actions](https://aws.amazon.com/blogs/machine-learning/deploy-ai-agents-on-amazon-bedrock-agentcore-using-github-actions/)
- [Create a CI/CD pipeline for Amazon ECS with GitHub Actions and AWS CodeBuild Tests](https://aws.amazon.com/blogs/containers/create-a-ci-cd-pipeline-for-amazon-ecs-with-github-actions-and-aws-codebuild-tests/)
- [What If we shipped models the same way we ship code?](https://www.cncf.io/blog/2026/03/27/the-weight-of-ai-models-why-infrastructure-always-arrives-slowly/)

```
└── <workspace>/
    ├── apps/
    │   ├── <app-name>/
    │   │   ├── app/
    │   │   ├── Dockerfile
    │   │   └── k8s/
    │   │       ├── deployment.yaml
    │   │       └── service.yaml
    │   └── <app-name>/
    │       ├── app/
    │       ├── Dockerfile
    │       └── k8s/
    │           ├── deployment.yaml
    │           └── service.yaml
    └── ingress/
        └── ingress.yaml         # Routes <workspace>.devailab.work/* → services (public)
```

```
infrastructure/
├── cloudflare/
│   ├── configmap.yaml       
│   ├── secret.yaml
│   └── tunnel.yaml
├── ssd-storageclass.yaml
└── namespaces.yaml              

spaces/
├── public/
│   ├── apps/
│   │   └── landing/
│   │       ├── app/
│   │       ├── Dockerfile
│   │       └── k8s/
│   │           ├── deployment.yaml
│   │           └── service.yaml
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
│   │   │       ├── deployment.yaml
│   │   │       └── service.yaml
│   │   ├── helloworld/
│   │   │   ├── app/
│   │   │   ├── Dockerfile
│   │   │   └── k8s/
│   │   │       ├── deployment.yaml
│   │   │       └── service.yaml
│   │   └── geminirealtime/
│   │       ├── app/
│   │       ├── Dockerfile
│   │       └── k8s/
│   │           ├── deployment.yaml
│   │           └── service.yaml
│   └── ingress/
│       └── ingress.yaml         # Routes mcd.devailab.work/* → services (auth required)
├── hackaton/
│   └── apps/
│      └── mcpserver/
│           ├── app/
│           │   ├── app.py
│           │   └── requirements.txt
│           ├── k8s/
│           │   ├── deployment.yaml
│           │   ├── service.yaml
│           │   └── Dockerfile
│           ├── ingress/
│           │   └── ingress.yaml     # Routes hackaton.devailab.work/mcp
│           └── nginx.conf
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



