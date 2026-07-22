# Day 1 - ACR, Docker, and Azure Web App for Containers

This repo is being used for Azure AI-200 exam prep and hands-on Azure deployment practice.

## Goal

Deploy a tiny containerized demo app to Azure by using:

- Azure CLI
- Docker
- Azure Container Registry (ACR)
- Azure App Service Plan
- Azure Web App for Containers

## Local Setup

Installed Azure CLI with Homebrew:

```bash
brew install azure-cli
```

Verified Azure CLI:

```bash
az version
```

Logged in to Azure:

```bash
az login
```

Listed resource groups:

```bash
az group list
```

Resource group used:

```text
az-ai-200
```

Location used:

```text
eastus
```

## Demo App

Downloaded a small container-ready Node.js sample app:

```bash
git clone https://github.com/Azure-Samples/acr-build-helloworld-node.git demo-acr-app
```

The app uses:

```text
server.js
Dockerfile
```

The container listens on port:

```text
80
```

Local Docker build:

```bash
docker build -t demo-acr-app:v1 .
```

Important Docker note:

```bash
docker build -t demo-acr-app:v1
```

failed because Docker needs a build context. The dot means "use the current directory as the build context":

```bash
docker build -t demo-acr-app:v1 .
```

## Azure Container Registry

Created an ACR named:

```text
gowthamdemoacr26
```

ACR login server:

```text
gowthamdemoacr26.azurecr.io
```

Enabled admin access:

```bash
az acr update --name gowthamdemoacr26 --admin-enabled true
```

Checked admin access:

```bash
az acr show --name gowthamdemoacr26 --query adminUserEnabled --output tsv
```

Logged Docker into ACR:

```bash
az acr login --name gowthamdemoacr26
```

## Local Build and Push to ACR

The correct image tag must include the ACR login server:

```bash
docker tag demo-acr-app:v1 gowthamdemoacr26.azurecr.io/demo-acr-app:v1
```

Push to ACR:

```bash
docker push gowthamdemoacr26.azurecr.io/demo-acr-app:v1
```

Important lesson:

```text
gowthamdemoacr26/demo-acr-app:v1
```

is treated as Docker Hub:

```text
docker.io/gowthamdemoacr26/demo-acr-app
```

For ACR, always include:

```text
gowthamdemoacr26.azurecr.io
```

## ACR Build / ACR Tasks

Tried ACR cloud build:

```bash
az acr build --registry gowthamdemoacr26 --image demo-acr-app:task-build-1.0 .
```

This failed with:

```text
TasksOperationsNotAllowed
```

Learning:

- `az acr build` uses ACR Tasks behind the scenes.
- Some Free Trial, Azure for Students, or restricted subscriptions do not allow ACR Tasks.
- A GitHub token does not bypass this restriction.
- Workaround: build locally or use GitHub Actions.

## GitHub Plan

Future CI/CD plan:

```text
GitHub push -> GitHub Actions -> docker build -> docker push to ACR
```

For GitHub Actions, store ACR credentials as GitHub repository secrets:

```text
ACR_USERNAME
ACR_PASSWORD
```

## App Service Plan

Created a Linux App Service Plan:

```bash
az appservice plan create --resource-group az-ai-200 --name asp-demo-acr --location eastus --is-linux --sku B1
```

Hit quota issues in `eastus`:

```text
Current Limit (Total VMs): 0
Amount required: 1
```

Learning:

- App Service Plans require compute quota.
- Some regions or subscriptions may have quota restrictions.
- Trying another region or requesting quota may be required.

The App Service Plan currently used:

```text
asp-demo-acr
```

Plan SKU:

```text
B1
```

Region:

```text
Central US
```

## Web App for Containers

Created a Web App for Containers:

```bash
az webapp create --resource-group az-ai-200 --plan asp-demo-acr --name demo-acr-webapp-gowtham --deployment-container-image-name gowthamdemoacr26.azurecr.io/demo-acr-app:v1
```

Configured the Web App to pull from ACR:

```bash
az webapp config container set --resource-group az-ai-200 --name demo-acr-webapp-gowtham --container-image-name gowthamdemoacr26.azurecr.io/demo-acr-app:v1 --container-registry-url https://gowthamdemoacr26.azurecr.io --container-registry-user gowthamdemoacr26 --container-registry-password "<acr-password>"
```

Set the container port:

```bash
az webapp config appsettings set --resource-group az-ai-200 --name demo-acr-webapp-gowtham --settings WEBSITES_PORT=80
```

Restarted the app:

```bash
az webapp restart --resource-group az-ai-200 --name demo-acr-webapp-gowtham
```

App URL:

```text
https://demo-acr-webapp-gowtham.azurewebsites.net
```

## Troubleshooting

### 503 Service Unavailable

The Web App initially returned:

```text
503 Service Unavailable
```

Checked logs:

```bash
az webapp log config --resource-group az-ai-200 --name demo-acr-webapp-gowtham --docker-container-logging filesystem
```

```bash
az webapp log tail --resource-group az-ai-200 --name demo-acr-webapp-gowtham
```

### ImagePullUnauthorizedFailure

Logs showed:

```text
ImagePullUnauthorizedFailure
Failed to pull image
Image pull failed with forbidden or unauthorized
```

Cause:

```text
App Service could not authenticate to ACR.
```

Fix:

- Confirm ACR admin user is enabled.
- Use the current ACR admin password.
- Re-run `az webapp config container set`.
- Restart the Web App.

### Wrong Tag

At one point the Web App pointed to:

```text
demo-acr-app:v1
```

but ACR only had:

```text
task-build-1.0
```

Checked tags:

```bash
az acr repository show-tags --name gowthamdemoacr26 --repository demo-acr-app --output table
```

Fix:

```bash
az webapp config container set --resource-group az-ai-200 --name demo-acr-webapp-gowtham --container-image-name gowthamdemoacr26.azurecr.io/demo-acr-app:task-build-1.0 --container-registry-url https://gowthamdemoacr26.azurecr.io --container-registry-user gowthamdemoacr26 --container-registry-password "$(az acr credential show --name gowthamdemoacr26 --query "passwords[0].value" --output tsv)"
```

### Exec Format Error

After credentials were fixed, logs showed:

```text
exec /usr/local/bin/docker-entrypoint.sh: exec format error
```

Cause:

```text
The image was built for ARM64 on Apple Silicon, but Azure App Service expects linux/amd64.
```

Fix:

```bash
docker buildx build --platform linux/amd64 --tag gowthamdemoacr26.azurecr.io/demo-acr-app:amd64-v1 --push .
```

Then update the Web App:

```bash
az webapp config container set --resource-group az-ai-200 --name demo-acr-webapp-gowtham --container-image-name gowthamdemoacr26.azurecr.io/demo-acr-app:amd64-v1 --container-registry-url https://gowthamdemoacr26.azurecr.io --container-registry-user gowthamdemoacr26 --container-registry-password "$(az acr credential show --name gowthamdemoacr26 --query "passwords[0].value" --output tsv)"
```

Restart:

```bash
az webapp restart --resource-group az-ai-200 --name demo-acr-webapp-gowtham
```

## Final Verification

The deployed app responds successfully:

```bash
curl -L https://demo-acr-webapp-gowtham.azurewebsites.net
```

Output:

```text
Hello World from ACR
Version: 15.14.0
```

## Key Exam Prep Takeaways

- ACR stores container images.
- Docker image names pushed to ACR must include the ACR login server.
- App Service Plan provides compute for Web Apps.
- Web App for Containers can pull an image from ACR.
- Private ACR pulls require authentication.
- `WEBSITES_PORT` tells App Service which port the container listens on.
- `az acr build` uses ACR Tasks.
- ACR Tasks can be restricted in free/student subscriptions.
- Apple Silicon builds may produce ARM64 images.
- Azure App Service for Linux commonly requires `linux/amd64` images.
- Use `docker buildx build --platform linux/amd64 --push` when deploying from Apple Silicon to App Service.

## Security Notes

- ACR admin passwords are secrets.
- Do not paste them into chat, docs, screenshots, or GitHub.
- If exposed, rotate both passwords:

```bash
az acr credential renew --name gowthamdemoacr26 --password-name password
```

```bash
az acr credential renew --name gowthamdemoacr26 --password-name password2
```
