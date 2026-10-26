# DevOps Assessment: Spring Boot + Docker + Azure DevOps Pipelines + Kubernetes + ArgoCD

A small Spring Boot REST API delivered through a full CI/CD and GitOps flow.

```
git push -> Azure Pipelines (build, test, push image, bump tag in k8s/deployment.yaml)
                                   |
                                   v
                          ArgoCD watches k8s/ (auto-sync) -> Kubernetes (2 replicas)
```

## Endpoints
| Path | Description |
|------|-------------|
| `/` | Static homepage that calls the health API |
| `/api/health` | JSON health check (status, version, pod name, timestamp) |

## Repository layout
```
src/                         Spring Boot source and tests
pom.xml                      Maven build (Spring Boot 2.7.18, Java 17)
Dockerfile, .dockerignore    Multi-stage, non-root image
azure-pipelines.yml          CI/CD pipeline (Azure DevOps)
k8s/                         Namespace, Deployment (2 replicas, probes), Service (NodePort)
argocd/application.yaml      ArgoCD Application with auto-sync, prune and self-heal
screenshots/                 Evidence images
```

## Prerequisites
Docker, kubectl, a Kubernetes cluster (Minikube shown), a Docker Hub account, a GitHub repo and an Azure DevOps organization. JDK 17 and Maven are optional locally, because the Docker build compiles the app inside the container.

## 1. Run locally
Option A: with Docker only (no Java or Maven needed)
```bash
docker build -t <DOCKERHUB_USER>/devops-demo:1.0.0 .
docker run --rm -p 8080:8080 <DOCKERHUB_USER>/devops-demo:1.0.0
curl http://localhost:8080/api/health
```
Option B: with JDK 17 and Maven installed
```bash
mvn spring-boot:run
```

## 2. Build and push the image manually (first time)
```bash
docker build -t <DOCKERHUB_USER>/devops-demo:1.0.0 .
docker login
docker push <DOCKERHUB_USER>/devops-demo:1.0.0
```

## 3. Configure the Azure DevOps pipeline
1. Push this repo to GitHub (branch `main`).
2. In Azure DevOps (dev.azure.com) create an organization and a project.
3. If the first run says "No hosted parallelism", request the free grant at https://aka.ms/azpipelines-parallelism-request, or use a self-hosted agent.
4. Create a Docker Hub access token (Account Settings > Security).
5. Project Settings > Service connections > New > Docker Registry > Docker Hub. Enter your username and token and name the connection `dockerhub-conn`.
6. Create a GitHub fine-grained personal access token with Contents: read and write on this repo. The pipeline uses it to commit the new image tag.
7. Replace the placeholders: `dockerHubUser` and `githubRepo` in `azure-pipelines.yml`, `YOUR_DOCKERHUB_USER` in `k8s/deployment.yaml`.
8. Pipelines > New pipeline > GitHub > select the repo > Existing Azure Pipelines YAML file > `/azure-pipelines.yml`. Approve the Azure Pipelines GitHub app when asked.
9. Before the first run: Edit > Variables > New variable `GITHUB_PAT`, paste the token, tick Keep this value secret.
10. Run the pipeline. After that, every push to `main` triggers it, and it can also be started manually with Run pipeline.

Each run builds and tests, pushes `<user>/devops-demo:<BuildId>` and `:latest`, then commits the new tag into `k8s/deployment.yaml`. No credentials are stored in the repo; they live in the service connection and a secret variable.

## 4. Deploy to Kubernetes (Minikube)
```bash
minikube start
kubectl apply -f k8s/
kubectl get pods -n devops-demo
kubectl get svc -n devops-demo
minikube service devops-demo -n devops-demo --url
```
On Kind or other clusters use `kubectl port-forward -n devops-demo svc/devops-demo 8081:80` and open http://localhost:8081.

## 5. GitOps with ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl port-forward svc/argocd-server -n argocd 8443:443
# initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
Edit `repoURL` in `argocd/application.yaml`, then:
```bash
kubectl apply -f argocd/application.yaml
```
Open https://localhost:8443 (user `admin`). The app should show **Synced** and **Healthy**. If the repo is private, add it under Settings > Repositories with a read-only token.

If you ran `kubectl apply -f k8s/` earlier, ArgoCD adopts the same resources without conflict.

## 6. Verify end to end
1. Change something small (for example the text in `src/main/resources/static/index.html`) and push to `main`.
2. Watch the Azure Pipelines run go green.
3. Watch the bot commit appear in `k8s/deployment.yaml`.
4. Watch ArgoCD auto-sync and roll the pods (`kubectl get pods -n devops-demo -w`).
5. Reload the homepage and call `/api/health`; the `pod` field changes between requests when load-balanced across replicas.

## Security notes
- No secrets in code or Git; `.gitignore` excludes env files, keys, logs and credential files.
- The container runs as a non-root user.
- Images are tagged with the commit SHA so every deployment is traceable and rollbacks are a Git revert.

## A note on Maven
Maven is the build tool the assignment requires. `pom.xml` is its config file, like a package list plus build instructions. You only meet it in two places: the pipeline runs `mvn clean verify`, and the Dockerfile runs `mvn package` inside the build stage. Nothing needs to be configured by hand.
