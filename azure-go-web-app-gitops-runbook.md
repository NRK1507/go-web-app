# Go Web Application --- Azure CI/CD & GitOps Runbook

**Project:** Containerized Go web application deployed to K3s on an
Azure VM\
**CI:** GitHub Actions \| **Registry:** Docker Hub \| **Packaging:**
Helm \| **CD:** Argo CD \| **Ingress:** NGINX

## 1. Project overview

This runbook documents the implementation completed for the Go web
application repository:

https://github.com/NRK1507/go-web-app

The project builds and tests a Go application, creates and publishes a
Docker image, updates a Helm chart with the image tag, and uses Argo CD
to synchronize that chart into a K3s cluster running on an Azure virtual
machine.

### Delivery flow

``` text
Developer push to GitHub (main)
              |
              v
       GitHub Actions
       - Go build and tests
       - golangci-lint
       - Docker build and push
       - Update Helm image tag in Git
              |
              +--------------------> Docker Hub
              |
              v
       GitHub Helm chart
              |
              v
           Argo CD
              |
              v
        K3s on Azure VM
              |
       Deployment / ReplicaSet
              |
             Pod
              |
        Go web application

Browser --> Azure NSG --> NGINX NodePort --> Ingress
                                           |
                                      Service --> Pod
```

## 2. Environment

  Component              Implemented configuration
  ---------------------- --------------------------------------
  Cloud                  Microsoft Azure
  VM OS                  Ubuntu 24.04 LTS
  VM size                Standard B2as v2 (2 vCPU, 8 GiB RAM)
  Kubernetes             Single-node K3s
  CI                     GitHub Actions
  Image registry         Docker Hub
  Deployment packaging   Helm
  GitOps CD              Argo CD
  Ingress                NGINX Ingress Controller

Use the current VM public IP from the Azure portal; public IPs can
change.

### Connect to the VM

``` bash
ssh -i /path/to/private-key.pem azureuser@<VM_PUBLIC_IP>
```

### Update Ubuntu

``` bash
sudo apt update
sudo apt upgrade -y
```

### Install Go locally (optional)

The project notes used Go 1.22.5:

``` bash
wget https://go.dev/dl/go1.22.5.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.22.5.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc
go version
```

The GitHub Actions runner also installs Go for CI. A local Go
installation is only needed when building or testing directly on the VM.

## 3. Repository structure

``` text
go-web-app/
├── .github/
│   └── workflows/
│       └── ci.yml
├── helm/
│   └── go-web-app-chart/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           └── ingress.yaml
├── static/
│   ├── images/
│   ├── about.html
│   ├── contact.html
│   ├── courses.html
│   └── home.html
├── Dockerfile
├── go.mod
├── main.go
└── main_test.go
```

The Go application serves static pages through routes including `/`,
`/courses`, `/about`, and `/contact`. The application listens on port
`8080` inside the container.

## 4. Build and test the application

``` bash
git clone https://github.com/NRK1507/go-web-app.git
cd go-web-app

go build -o go-web-app .
go test ./...
```

Run locally:

``` bash
./go-web-app
```

The application listens on port `8080`.

## 5. Containerize with Docker

The project uses a multi-stage Docker build: a Go builder stage compiles
the binary, and a Distroless runtime stage contains the binary and
static assets.

Representative Dockerfile:

``` dockerfile
FROM golang:1.22 AS build
WORKDIR /app

COPY go.mod ./
RUN go mod download

COPY . .
RUN go build -o main .

FROM gcr.io/distroless/base
WORKDIR /
COPY --from=build /app/main ./main
COPY --from=build /app/static ./static

EXPOSE 8080
CMD ["./main"]
```

Build and run:

``` bash
docker build -t go-web-app:v1 .
docker run -d --name go-web-app -p 8080:8080 go-web-app:v1
```

Publish manually (the automated pipeline uses a run ID tag):

``` bash
docker tag go-web-app:v1 <DOCKERHUB_USERNAME>/go-web-app:v1
docker login
docker push <DOCKERHUB_USERNAME>/go-web-app:v1
```

**Credential handling:** Never commit Docker Hub tokens, GitHub tokens,
SSH private keys, passwords, or kubeconfig credentials. Store CI
credentials in GitHub Actions secrets.

## 6. Install and verify K3s

``` bash
curl -sfL https://get.k3s.io | sh -
sudo kubectl get nodes -o wide
```

K3s commonly installs Traefik as its default Ingress controller. This
project also installed NGINX Ingress and used the `nginx` IngressClass
for the application. When multiple controllers exist, ensure each
Ingress resource specifies the intended class.

Do not make the K3s kubeconfig world-readable. Protect copied kubeconfig
files as credentials.

## 7. Helm chart

Chart location:

``` text
helm/go-web-app-chart/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

Relevant `values.yaml` settings (replace placeholders with the real
registry and tag):

``` yaml
replicaCount: 1

image:
  repository: <DOCKERHUB_USERNAME>/go-web-app
  pullPolicy: IfNotPresent
  tag: "<IMAGE_TAG>"

ingress:
  enabled: true
  className: nginx
  annotations: {}
  hosts:
    - host: go-web-app
      paths:
        - path: /
          pathType: Prefix
```

The Service exposes port `80` and targets container port `8080`. The
Ingress routes host `go-web-app` to that Service.

Validate the chart:

``` bash
helm lint ./helm/go-web-app-chart
helm template go-web-app ./helm/go-web-app-chart
```

Useful inspection commands:

``` bash
helm list -A
helm status go-web-app
kubectl get all -n default
```

Argo CD is the owner of the GitOps deployment. Avoid making manual Helm
changes that conflict with Argo CD reconciliation.

## 8. GitHub Actions CI pipeline

Workflow file:

``` text
.github/workflows/ci.yml
```

The workflow supports manual execution through `workflow_dispatch` and
push-triggered execution on `main` (subject to the workflow's configured
path filters).

### Pipeline stages

  -------------------------------------------------------------------------
  Job                                 Responsibility
  ----------------------------------- -------------------------------------
  Build and test                      Check out source, set up Go, compile
                                      the app, run unit tests

  Code quality                        Run `golangci-lint`

  Docker build and push               Build with Docker Buildx and push to
                                      Docker Hub

  Update Helm chart                   Write the new image tag into
                                      `helm/go-web-app-chart/values.yaml`
                                      and commit the change
  -------------------------------------------------------------------------

### Required GitHub Actions secrets

Configure these in **Repository → Settings → Secrets and variables →
Actions**:

  -----------------------------------------------------------------------
  Secret                              Purpose
  ----------------------------------- -----------------------------------
  `DOCKERHUB_USERNAME`                Docker Hub username

  `DOCKERHUB_TOKEN`                   Docker Hub access token

  `TOKEN`                             GitHub token used by the workflow
                                      to commit the Helm tag update, if
                                      configured that way
  -----------------------------------------------------------------------

Use least-privilege permissions. A repository-scoped token with Contents
read/write access is preferable to a broadly privileged token.

### Image versioning

The pipeline tags images using:

``` text
<DOCKERHUB_USERNAME>/go-web-app:<GITHUB_RUN_ID>
```

The same run ID is written to the Helm chart's `image.tag`. Ensure the
`image.repository` in `values.yaml` matches the repository to which the
workflow pushes.

Prevent the Helm-update commit from causing an unintended CI loop. The
workflow can use an appropriate path filter or a `[skip ci]` commit
message; ensure the selected method matches the actual trigger
configuration.

### Verify CI

1.  Open the GitHub repository and select **Actions**.
2.  Open the latest workflow run.
3.  Verify build, tests, code quality, Docker push, and Helm update.
4.  Confirm the Helm tag update was committed.

## 9. Install and expose Argo CD

Create the namespace and install Argo CD:

``` bash
kubectl create namespace argocd

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

If the CRD install fails due to the client-side apply annotation-size
limit, use server-side apply:

``` bash
kubectl apply --server-side -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

If server-side apply reports field ownership conflicts, inspect the
affected resources and managed fields before considering
`--force-conflicts`. Do not force ownership changes without
understanding the existing field manager.

Verify:

``` bash
kubectl get pods -n argocd
```

### Configure Argo CD as NodePort

This project used NodePort on the single-node K3s cluster:

``` bash
kubectl patch svc argocd-server -n argocd \
  -p '{"spec":{"type":"NodePort"}}'

kubectl get svc argocd-server -n argocd
```

Use the HTTPS NodePort returned by Kubernetes; do not assume it is
fixed. Add an Azure NSG inbound rule for that TCP port, restricted to
your trusted client IP.

Access:

``` text
https://<VM_PUBLIC_IP>:<ARGOCD_HTTPS_NODEPORT>
```

A browser certificate warning may appear with the default self-signed
certificate in a lab environment.

### Initial admin login

Username:

``` text
admin
```

Retrieve the initial password:

``` bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

Change the initial password and protect it. Do not place credentials in
this runbook or in Git.

## 10. Create the Argo CD Application

Create an Application in the Argo CD UI.

### General

  Field              Value
  ------------------ --------------
  Application name   `go-web-app`
  Project            `default`
  Sync policy        Automatic
  Auto-Sync          Enabled
  Prune resources    Enabled
  Self Heal          Enabled

### Source

  Field            Value
  ---------------- -----------------------------------------
  Repository URL   `https://github.com/NRK1507/go-web-app`
  Revision         `HEAD`
  Path             `helm/go-web-app-chart`

### Destination

  Field         Value
  ------------- ----------------------------------
  Cluster URL   `https://kubernetes.default.svc`
  Namespace     `default`

Argo CD reads the Helm chart from Git and reconciles the rendered
resources in K3s. With automated sync enabled, a committed Helm values
change can deploy a new image without a manual `helm upgrade`.

Verify:

``` bash
kubectl get pods -n default
kubectl get deployment go-web-app -n default
kubectl get svc go-web-app -n default
kubectl get ingress -n default
```

Target Argo CD status:

-   Sync: `Synced`
-   Health: `Healthy`

## 11. NGINX Ingress and external access

Inspect the controller and Ingress:

``` bash
kubectl get ingressclass
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
kubectl get ingress -A
```

The NGINX controller Service was changed to NodePort because this
self-managed K3s cluster did not have an Azure Load Balancer provisioned
for the Service:

``` bash
kubectl patch svc ingress-nginx-controller -n ingress-nginx \
  -p '{"spec":{"type":"NodePort"}}'

kubectl get svc -n ingress-nginx
```

Use the actual NodePort shown by `kubectl get svc`. During this project,
the HTTP NodePort was `31767`.

### Host-based routing

The Ingress rule uses host `go-web-app`. A request to the VM IP alone
does not match that host rule.

On the Windows client, edit this file as Administrator:

``` text
C:\Windows\System32\drivers\etc\hosts
```

Add:

``` text
<VM_PUBLIC_IP> go-web-app
```

Then browse to:

``` text
http://go-web-app:<NGINX_HTTP_NODEPORT>/courses
```

The tested URL was:

``` text
http://go-web-app:31767/courses
```

This hosts-file mapping is a local lab workaround, not public DNS.

### Verify from the VM

``` bash
curl -i -H "Host: go-web-app" \
  http://localhost:<NGINX_HTTP_NODEPORT>/courses
```

The successful test returned:

``` text
HTTP/1.1 200 OK
```

This verifies NGINX Ingress → Service → application Pod routing.

## 12. End-to-end GitOps deployment test

Use this test to prove that a source change reaches the running
application without a manual Helm upgrade.

1.  Make a visible change to an HTML file under `static/`, for example
    the `<h1>` heading in `static/courses.html`.
2.  Commit and push to `main`.
3.  Verify the GitHub Actions workflow completes successfully.
4.  Confirm a new Docker image is published.
5.  Confirm `helm/go-web-app-chart/values.yaml` contains the new image
    tag.
6.  Verify Argo CD detects and synchronizes the new Git revision.
7.  Inspect the Deployment and Pod:

``` bash
kubectl get deployment go-web-app
kubectl get pods -l app=go-web-app -o wide

kubectl get deployment go-web-app \
  -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
```

8.  Verify the new HTML content is served:

``` bash
curl -s -H "Host: go-web-app" \
  http://localhost:<NGINX_HTTP_NODEPORT>/courses \
  | grep -i "your updated heading"
```

9.  Refresh the browser and verify the visible change.

**HTML note:** Changing `<title>` changes the browser tab title. It does
not change the visible page heading. Update `<h1>` to test a visible
heading change.

The completed project test verified that the updated image tag reached
the Kubernetes Deployment, the changed HTML title was served by the
application, and the NGINX Ingress route returned HTTP 200.

## 13. Troubleshooting reference

### General Kubernetes state

``` bash
kubectl get nodes -o wide
kubectl get all -n default
kubectl get pods -A
kubectl describe pod <POD_NAME> -n default
kubectl logs <POD_NAME> -n default
kubectl get events -n default --sort-by=.metadata.creationTimestamp
```

### Deployment and image

``` bash
kubectl get deployment go-web-app -o wide
kubectl describe deployment go-web-app
kubectl get rs -l app=go-web-app
kubectl get pods -l app=go-web-app -o wide
```

### Compare Helm and deployed image tags

``` bash
grep -A4 '^image:' helm/go-web-app-chart/values.yaml

kubectl get deployment go-web-app \
  -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
```

If the tags differ, inspect Argo CD sync status and the rendered Helm
manifest before changing resources manually.

### Argo CD

``` bash
kubectl get pods -n argocd
kubectl get svc -n argocd
```

Use the Argo CD UI to inspect the application resource tree, sync
status, health, events, and sync history.

### NGINX Ingress

``` bash
kubectl get ingressclass
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
kubectl get ingress -A
kubectl describe ingress go-web-app -n default
```

## 14. Azure networking and security

Azure NSG rules and Kubernetes NodePorts are separate layers. Traffic
must be permitted by the NSG and reach the NodePort on the VM.

Ports observed during this project:

  ------------------------------------------------------------------------
  Purpose                                       Port Notes
  --------------------- ---------------------------- ---------------------
  Direct application                     TCP `32446` Application Service
  access                                             NodePort used during
                                                     testing

  NGINX Ingress HTTP                     TCP `31767` Host-based Ingress
                                                     access

  Argo CD HTTPS                          TCP `30515` Argo CD NodePort
                                                     observed during
                                                     setup; verify current
                                                     value
  ------------------------------------------------------------------------

NodePort assignments can change if Services are recreated. Always verify
them with `kubectl get svc`.

Security recommendations:

-   Restrict NSG source addresses to trusted client IPs.
-   Do not expose Argo CD administration publicly without additional
    access controls.
-   Use a real DNS name and trusted TLS certificate for production.
-   Rotate initial credentials and use least-privilege tokens.
-   Keep private keys, tokens, kubeconfig files, and passwords out of
    Git.
-   Review Azure VM, disk, public IP, and networking costs when the lab
    is idle.

## 15. Project outcome and portfolio description

### Capabilities demonstrated

-   Go application build and unit testing.
-   Multi-stage Docker image creation.
-   Docker Hub image publishing.
-   K3s and Kubernetes operations.
-   Deployments, ReplicaSets, Services, and Ingress.
-   Helm chart packaging and values-based configuration.
-   GitHub Actions CI automation.
-   Argo CD GitOps continuous delivery.
-   NGINX host-based routing.
-   Azure VM networking and NSG configuration.
-   End-to-end deployment validation and troubleshooting.

### Portfolio summary

**Go Web Application --- Automated CI/CD and GitOps on Azure**

Implemented an end-to-end CI/CD and GitOps workflow for a containerized
Go web application on an Azure Linux VM running K3s. Built and tested
the application with GitHub Actions, published versioned Docker images
to Docker Hub, and automated Helm chart image-tag updates. Configured
Argo CD to synchronize Git-managed Helm releases into Kubernetes and
exposed the application through NGINX Ingress. Validated application
rollouts and HTTP responses without requiring manual Helm upgrades for
subsequent releases.

**Technologies:** Azure VM, Ubuntu, Go, Docker, Docker Hub, K3s,
Kubernetes, Helm, GitHub Actions, Argo CD, NGINX Ingress, Git.

## 16. Further improvements before calling it production-grade

This is a valuable hands-on portfolio project, but it is a single-node
lab environment, not a highly available production platform. Potential
next improvements:

-   Configure real DNS and trusted TLS certificates.
-   Harden Argo CD access and rotate credentials.
-   Add CPU/memory requests and limits and an HPA where appropriate.
-   Add monitoring and centralized logging.
-   Add image scanning and dependency/security checks to CI.
-   Document rollback and recovery procedures.
-   Back up important cluster and application state.
-   Consider a multi-node or managed Kubernetes architecture when high
    availability is required.

Only claim these capabilities after implementing and testing them.
