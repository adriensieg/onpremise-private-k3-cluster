
# Situation

- arm64 or amd64
- arm64 or x64
- Should I store my secrets into Deployment.yaml? Where should i store my .env secrets?

## Implementation

### How it works? 

- **GitHub Actions** builds & pushes an **ARM64 Docker image** to **GHCR**
- *ArgoCD* detects the new image tag and syncs your **K8s manifests**. ArgoCD runs inside our cluster and pulls from GitHub. It never needs inbound internet access

- **K3s** deploys the new pods
- **NGINX** + **Cloudflare** serve it publicly

Why commit the manifest back? ArgoCD watches your Git repo for manifest changes. When GitHub Actions updates the image tag in deployment.yaml and pushes that commit, ArgoCD detects the diff and automatically syncs the new deployment to your cluster. This is the GitOps pattern — Git is the single source of truth.

You can push from anywhere — your laptop at a café, your office, anywhere with internet. ArgoCD on your cluster polls GitHub every 3 minutes (or you can configure a webhook for instant sync).

### What we must implement? 
- **Install ArgoCD on our K3s cluster**: SSH into our control plane (we need to be on our local network for this initial setup)
- **Restructure our GitHub repository**: Our repo needs a clear separation between **app source code** and **Kubernetes manifests**. ArgoCD watches the manifests; GitHub Actions updates them.
- **Configure GHCR access**: Our cluster needs credentials to **pull images from GHCR**. Create a **GitHub Personal Access Token (PAT)** with `read:packages` scope at `https://github.com/settings/tokens`. Then create the **pull secret in each namespace** that needs it. 
- **GitHub Actions workflow**: Create `.github/workflows/deploy.yaml`. This workflow detects which app changed and only rebuilds that app.
- **Configure ArgoCD Applications**: Create **one ArgoCD Application resource per workspace**. ArgoCD will watch that folder in our repo and **keep the cluster in sync**.
- **Repository secrets**: That's it. `GITHUB_TOKEN` is automatically injected by **Actions** and has `write` access to **GHCR for our repo**. No additional secrets needed for the basic flow.
- The **new deployment workflow**
- **Monitoring deployments**

### Delete all `application` namespaces

```
kubectl delete namespace public perso
kubectl get namespaces -w
```

- Verify infrastructure **namespaces are intact**

```
kubectl get namespaces
# cloudflare        Active    ← cloudflared tunnel still running
# ingress-nginx     Active    ← NGINX ingress controller still running
# kube-system       Active    ← K3s system pods
```

- Recreate the namespaces fresh
```
kubectl apply -f infrastructure/namespaces.yaml
```

### Restructure the repository

#### Create the new top-level folders

Our repo currently has 
- `infrastructure/`
- `spaces/`

We now need:
- `.github`
- `argocd`

#### The overall layout: 

```
onpremise-private-k3-cluster/
├── .github/
│   └── workflows/
│       └── deploy.yaml              ← GitHub Actions (new)
├── argocd/
│   └── apps/
│       ├── public.yaml              ← ArgoCD Application (new)
│       ├── mcd.yaml                 ← ArgoCD Application (new)
│       ├── perso.yaml               ← ArgoCD Application (new)
│       ├── hackaton.yaml            ← ArgoCD Application (new)
│       └── techie.yaml              ← ArgoCD Application (new)
├── infrastructure/                  ← unchanged
│   ├── cloudflare/
│   │   ├── configmap.yaml       
│   │   ├── secret.yaml
│   │   └── tunnel.yaml
│   ├── ssd-storageclass.yaml
│   └── namespaces.yaml
└── spaces/                          ← structure unchanged, image refs updated
    ├── public/
    │   └── k8s/deployment.yaml      ← image: ghcr.io/adriensieg/public-landing:sha
    ├── mcd/apps/{landing,helloworld,geminirealtime}/
    │   └── k8s/deployment.yaml      ← image: ghcr.io/adriensieg/mcd-*:sha
    ├── hackaton/apps/mcpserver/
    │   └── k8s/deployment.yaml      ← image: ghcr.io/adriensieg/hackaton-mcpserver:sha
    └── .../...
```

- Each public application must be structured in this way:

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

- A private authenticated workspace must be organized in this way:
```
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
│   │   ├── helloworld/
│   │   └── geminirealtime/
│   └── ingress/
│       └── ingress.yaml         # Routes mcd.devailab.work/* → services (auth required)
```

#### Update deployment.yaml image references

1. Every `deployment.yaml` currently references a **local image** name like `landing:latest`. We must change **each one** to its **GHCR equivalent**. `image: ghcr.io/adriensieg/public-landing:latest`
2. <mark>The **workspace-appname prefix** is important — it avoids **name collisions** in GHCR since all images live under the same account. </mark> `image: ghcr.io/adriensieg/<workspace>-<application>:latest`
3. `imagePullPolicy`: `Never` must be **`Always`**
4. Also add `imagePullSecrets` to each deployment spec.

<img width="70%" height="70%" alt="image" src="https://github.com/user-attachments/assets/b32948a2-29ef-4378-8b1d-0169f378c4b1" />

Add secret to each deployment.
```
spec:
  template:
    spec:
      containers:
        - name: landing
          image: ghcr.io/adriensieg/public-landing:latest
          imagePullPolicy: Always
      imagePullSecrets:
        - name: ghcr-secret    # ← add this to every deployment
```

#### 3. Create the GitHub Actions workflow

Create `.github/workflows/deploy.yaml`

### 4. Create the ArgoCD Application manifests

Create **one file per workspace**. These tell ArgoCD **which folder to watch** and **which namespace to deploy into**.
- `argocd/apps/public.yaml`
- `argocd/apps/mcd.yaml`
- `argocd/apps/perso.yaml`
- ...

###  Install ArgoCD on the cluster (must be home)

▶️ SSH into the control plane for these command - Install ArgoCD
```
# On your LOCAL machine (PowerShell) — opens SSH session
ssh -i $HOME\.ssh\k3s-cluster adsieg@10.0.0.50

# Now on the control plane node:
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all pods to be ready (takes 2-3 minutes)
kubectl wait --for=condition=Ready pod --all -n argocd --timeout=300s
```

▶️  Install the ArgoCD CLI on your local Windows machine
```
winget install ArgoProj.ArgoCD
```

▶️ Verify it works:
```
argocd version --client
```

▶️ Get the initial admin password
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

▶️ Log in to ArgoCD from our local machine
- Open a second PowerShell window (keep the SSH session open). Forward the ArgoCD port:
```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

- In a third PowerShell window, log in via CLI:

```
argocd login localhost:8080 `
  --username admin `
  --password <PASTE_PASSWORD_HERE> `
  --insecure
```

argocd login localhost:8080 --username admin --password hGjYz4DgcRcvrAC5 --insecure


▶️ Change the default password immediately
```
argocd account update-password
```

### Configure GHCR access (must be home)

#### Create a GitHub Personal Access Token

Go to https://github.com/settings/tokens → "Generate new token (classic)".
Settings:
- Note: k3s-ghcr-pull
- Expiration: No expiration (acceptable for a homelab read-only token)
- Scope: check only read:packages

#### Create the pull secret in every namespace
Run this from our local PowerShell (kubectl must be configured, which it already is):

```
$PAT = "ghp_PASTE_YOUR_TOKEN_HERE"
$USER = "adriensieg"
$EMAIL = "adriensieg@hotmail.fr"

foreach ($NS in @("public","perso")) {
    kubectl create secret docker-registry ghcr-secret `
      --docker-server=ghcr.io `
      --docker-username=$USER `
      --docker-password=$PAT `
      --docker-email=$EMAIL `
      -n $NS
    Write-Host "Created ghcr-secret in namespace $NS"
}
```

Verify:
```
foreach ($NS in @("public", "perso")) {
    kubectl get secret ghcr-secret -n $NS
}
```

### Connect ArgoCD to your GitHub repository (must be home)

#### 1. Add the repo to ArgoCD
Our repo is public, so no credentials are needed for ArgoCD to read it. 
If we ever make it private, run this instead and use a PAT with repo scope:

```
# For a public repo — no auth needed:
argocd repo add https://github.com/adriensieg/onpremise-private-k3-cluster-raspberry

# For a private repo — use this instead:
# argocd repo add https://github.com/adriensieg/onpremise-private-k3-cluster-raspberry `
#   --username adriensieg `
#   --password ghp_YOUR_PAT_WITH_REPO_SCOPE
```

#### 2. Apply the ArgoCD Application manifests

This registers each workspace with ArgoCD so it starts watching our repo:
```
kubectl apply -f argocd/apps/public.yaml
kubectl apply -f argocd/apps/perso.yaml
```

Check that ArgoCD sees them:
```
argocd app list
```

We should see all 5 apps listed with status Synced or OutOfSync. OutOfSync is normal at this point — it means the manifests in Git reference GHCR images that haven't been pushed yet.

### Seed the registry with initial images (must be home)

This is the one-time migration step. You need to push the current images to GHCR so ArgoCD has something real to deploy. After this, every future deploy is fully automated.

```
$PAT = "ghp_PASTE_YOUR_TOKEN_HERE"
echo $PAT | docker login ghcr.io -u adriensieg --password-stdin
```


# Architecture

<img width="75%" height="75%" alt="image" src="https://github.com/user-attachments/assets/b870d09e-b42a-43ab-ae68-795f98d6a8f6" />


<img width="800" height="370" alt="image" src="https://github.com/user-attachments/assets/fb58951a-e74f-4a30-b4d8-22633305aa7c" />


<img width="1646" height="986" alt="image" src="https://github.com/user-attachments/assets/09d46404-adcd-487e-89af-d0b2691cbb54" />


# Bibliography

- [Deploy AI agents on Amazon Bedrock AgentCore using GitHub Actions](https://aws.amazon.com/blogs/machine-learning/deploy-ai-agents-on-amazon-bedrock-agentcore-using-github-actions/)
- [Create a CI/CD pipeline for Amazon ECS with GitHub Actions and AWS CodeBuild Tests](https://aws.amazon.com/blogs/containers/create-a-ci-cd-pipeline-for-amazon-ecs-with-github-actions-and-aws-codebuild-tests/)
- [What If we shipped models the same way we ship code?](https://www.cncf.io/blog/2026/03/27/the-weight-of-ai-models-why-infrastructure-always-arrives-slowly/)

  

