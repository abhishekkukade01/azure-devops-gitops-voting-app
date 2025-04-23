# Azure GitOps Voting App

This project demonstrates a full CI/CD pipeline using Azure DevOps and Argo CD for a microservice-based application deployed to Azure Kubernetes Service (AKS).

The app consists of three microservices:

| Microservice | Language | Role                            |
|--------------|----------|----------------------------------|
| voting       | Python   | User voting interface            |
| worker       | .NET     | Processes votes from Redis       |
| result       | Node.js  | Displays vote results from Postgres |

---

## 📦 Tech Stack

- Azure DevOps (CI/CD Pipelines)
- Docker
- Azure Container Registry (ACR)
- Azure Kubernetes Service (AKS)
- Argo CD (GitOps-based deployment)
- Kubernetes

---

## 🚀 Project Architecture

![image](https://github.com/user-attachments/assets/ade7b319-dc76-418f-93dc-7f33bbcd242d)


---

## 🛠️ Getting Started

### 1. Prerequisites

- Azure subscription
- Azure DevOps organization and project
- Azure Kubernetes Service (AKS) cluster
- Azure Container Registry (ACR)
- GitOps repo for Kubernetes manifests

---

### 2. Clone and Import Project

- Clone: https://github.com/dockersamples/example-voting-app
- Import into Azure DevOps Repos

---

### 3. CI Setup (Azure Pipelines)

Each microservice has its own Azure DevOps YAML pipeline:

- Builds Docker image
- Pushes to ACR
- Updates image tag in deployment.yaml in GitOps repo

CI pipelines:

| Service  | Pipeline File              |
|----------|----------------------------|
| voting   | azure-pipelines-voting.yml |
| result   | azure-pipelines-result.yml |
| worker   | azure-pipelines-worker.yml |

Each pipeline:
- Triggers on changes in its folder
- Tags image using Build ID
- Updates the corresponding manifest in gitops-infra repo

---

### 4. GitOps Repo Structure

Create a separate Git repo (e.g. gitops-infra) with the following structure:

gitops-infra/ ├── voting/ │ └── deployment.yaml ├── result/ │ └── deployment.yaml ├── worker/ │ └── deployment.yaml


Each deployment.yaml points to the correct Docker image:

yaml
containers:
  - name: voting
    image: <your-acr>.azurecr.io/voting-app:<tag>

## 5. Click **Sync** to Deploy
* Click on the app → then click **SYNC**
* Argo CD will:
  * Read the `deployment.yaml`
  * Deploy the image to AKS from ACR
  * Show status as **Healthy** and **Synced**



Each app will have its own Argo CD instance that tracks one folder in the Git repo and deploys that microservice.

## Enable Auto-Sync (Optional)
To fully automate:
1. Go into any application
2. Click **App Details** → **Edit**
3. Enable:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

## Step 6: Auto-Update Kubernetes YAML with New Image Tags

### Problem:
Your CI pipeline builds a new image (e.g., `voting-app:abc123`) but your `deployment.yaml` in Git still says: `voting-app:latest` → Argo CD sees *no change* in Git → No redeploy happens

### Solution:
After building a new image in Azure Pipelines:
1. **Update the image tag** in the `deployment.yaml` file
2. **Commit and push** that file back to your GitOps repo

This will trigger Argo CD to auto-sync and deploy the new version.

## Step-by-Step: Auto-update `deployment.yaml` in CI Pipeline

Let's take `voting` service as the example.

### A. Modify CI Pipeline (`azure-pipelines-voting.yml`)
Replace your existing YAML with this version:

```yaml
trigger:
  paths:
    include:
    - voting/**

pool:
  name: MySelfHostedPool

variables:
  imageName: voting-app
  acrLoginServer: <your-acr-name>.azurecr.io
  gitopsRepo: https://<your-devops-org>@dev.azure.com/<your-org>/<project>/_git/gitops-infra
- group: acr-credentials

stages:
- stage: BuildPush
  jobs:
  - job: BuildPushImage
    steps:
    - script: |
        echo "🛠️ Building Docker image..."
        IMAGE_TAG=$(Build.BuildId)
        docker build -t $(imageName):$IMAGE_TAG ./voting
        docker tag $(imageName):$IMAGE_TAG $(acrLoginServer)/$(imageName):$IMAGE_TAG
        docker push $(acrLoginServer)/$(imageName):$IMAGE_TAG
        echo "##vso[task.setvariable variable=IMAGE_TAG]$IMAGE_TAG"
      displayName: "Build and Push Image"

- stage: UpdateManifest
  dependsOn: BuildPush
  jobs:
  - job: UpdateDeploymentYAML
    steps:
    - script: |
        git config --global user.email "ci-bot@devops.com"
        git config --global user.name "Azure CI Bot"
        git clone $(gitopsRepo)
        cd gitops-infra/voting
        echo "🔁 Updating deployment.yaml with image tag: $(IMAGE_TAG)"
        sed -i "s|$(imageName):.*|$(imageName):$(IMAGE_TAG)|" deployment.yaml
        git add deployment.yaml
        git commit -m "Update image tag to $(IMAGE_TAG)"
        git push
      displayName: "Update deployment.yaml in GitOps repo"
```
### B. Ensure Pipeline Has Repo Write Access
You'll need to create a **Personal Access Token (PAT)** in Azure DevOps and add it as a secret.
1. Go to **User Settings → Personal Access Tokens**
2. Create token with:
   * Repo → **Read & Write**
   * Expiry: 30 days
3. Add it to Azure Pipelines as a variable group:
   * `GITOPS_PAT` → your token (mark as secret)

In pipeline, replace `$(gitopsRepo)` with:

```bash
https://$(GITOPS_PAT)@dev.azure.com/<org>/<project>/_git/gitops-infra
````

### C. Update Argo CD to Use Dynamic Tags (optional)
In `deployment.yaml`, switch from `latest` to something like:

```yaml
image: <your-acr>.azurecr.io/voting-app:1234
````
The pipeline will replace that with the real tag on each update.
Final Result:

Every new image tag will automatically:

Update the GitOps repo
Trigger Argo CD
Deploy the latest version to AKS
