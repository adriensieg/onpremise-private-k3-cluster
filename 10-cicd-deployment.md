
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

### 0. Delete all application namespaces

```
kubectl delete namespace public mcd perso hackaton techie
kubectl get namespaces -w
```

- Verify infrastructure namespaces are intact

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

### 1. Restructure the repository 

- The overall layout: 

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

### 2. Update deployment.yaml image references

Every `deployment.yaml` currently references a **local image** name like `landing:latest`. You must change each one to its **GHCR equivalent**. 

<img width="75%" height="75%" alt="image" src="https://github.com/user-attachments/assets/e191916f-d4f5-458d-9772-0aee9cbb0987" />

For example: 
```
spaces/public/apps/landing/k8s/deployment.yaml
  image: ghcr.io/adriensieg/public-landing:latest
```

<mark>The workspace-appname prefix is important — it avoids name collisions in GHCR since all images live under the same account. </mark> `image: ghcr.io/adriensieg/<workspace>-<application>:latest`

Also add imagePullSecrets to each deployment spec:

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

### 3. Create the GitHub Actions workflow

```
name: Build and deploy

on:
  push:
    branches: [main]
    paths:
      - 'spaces/**/app/**'
      - 'spaces/**/Dockerfile'

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.build-matrix.outputs.matrix }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2

      - name: Build changed-app matrix
        id: build-matrix
        run: |
          CHANGED=$(git diff --name-only HEAD~1 HEAD)
          APPS="[]"

          for WORKSPACE in public mcd perso hackaton techie; do
            for APP_DIR in spaces/$WORKSPACE/apps/*/; do
              [ -d "$APP_DIR" ] || continue
              APP=$(basename "$APP_DIR")
              if echo "$CHANGED" | grep -qE "^spaces/$WORKSPACE/apps/$APP/(app/|Dockerfile)"; then
                ENTRY="{\"workspace\":\"$WORKSPACE\",\"app\":\"$APP\",\"image\":\"$WORKSPACE-$APP\"}"
                APPS=$(echo "$APPS" | jq ". + [$ENTRY]")
              fi
            done
          done

          echo "matrix={\"include\":$(echo $APPS)}" >> $GITHUB_OUTPUT

  build-and-push:
    needs: detect-changes
    if: ${{ needs.detect-changes.outputs.matrix != '{"include":[]}' }}
    runs-on: ubuntu-latest
    strategy:
      matrix: ${{ fromJson(needs.detect-changes.outputs.matrix) }}
    permissions:
      contents: write
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push ARM64 image
        uses: docker/build-push-action@v5
        with:
          context: spaces/${{ matrix.workspace }}/apps/${{ matrix.app }}
          platforms: linux/arm64
          push: true
          tags: |
            ghcr.io/adriensieg/${{ matrix.image }}:latest
            ghcr.io/adriensieg/${{ matrix.image }}:${{ github.sha }}

      - name: Update image tag in manifest
        run: |
          # Find the deployment yaml (handles both naming conventions)
          MANIFEST=$(find spaces/${{ matrix.workspace }}/apps/${{ matrix.app }}/k8s/ \
            -name "*deployment.yaml" | head -1)
          
          sed -i \
            "s|image: ghcr.io/adriensieg/${{ matrix.image }}:.*|image: ghcr.io/adriensieg/${{ matrix.image }}:${{ github.sha }}|" \
            "$MANIFEST"
          
          echo "Updated $MANIFEST"

      - name: Commit updated manifest
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add spaces/${{ matrix.workspace }}/apps/${{ matrix.app }}/k8s/
          git diff --staged --quiet || git commit -m \
            "ci: ${{ matrix.workspace }}/${{ matrix.app }} → ${{ github.sha }}"
          git push
```

# Architecture

<img width="75%" height="75%" alt="image" src="https://github.com/user-attachments/assets/b870d09e-b42a-43ab-ae68-795f98d6a8f6" />


<img width="800" height="370" alt="image" src="https://github.com/user-attachments/assets/fb58951a-e74f-4a30-b4d8-22633305aa7c" />


<img width="1646" height="986" alt="image" src="https://github.com/user-attachments/assets/09d46404-adcd-487e-89af-d0b2691cbb54" />


# Bibliography

- [Deploy AI agents on Amazon Bedrock AgentCore using GitHub Actions](https://aws.amazon.com/blogs/machine-learning/deploy-ai-agents-on-amazon-bedrock-agentcore-using-github-actions/)
- [Create a CI/CD pipeline for Amazon ECS with GitHub Actions and AWS CodeBuild Tests](https://aws.amazon.com/blogs/containers/create-a-ci-cd-pipeline-for-amazon-ecs-with-github-actions-and-aws-codebuild-tests/)
- [What If we shipped models the same way we ship code?](https://www.cncf.io/blog/2026/03/27/the-weight-of-ai-models-why-infrastructure-always-arrives-slowly/)

  

