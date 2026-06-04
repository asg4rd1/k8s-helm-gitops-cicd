# kubernetes-helm-gitops

End-to-end GitOps project: a Flask app containerized with Docker, deployed to Kubernetes via Helm, and automated with GitHub Actions.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [Minikube](https://minikube.sigs.k8s.io/docs/start/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/)

---

## Phase 1 — Docker

Build the app image from the `app/` directory:

```bash
docker build -t flask-app:v1 .
```

Verify it works locally:

```bash
docker run -p 5000:5000 flask-app:v1
curl http://localhost:5000
# {"status": "ok"}
```

---

## Phase 2 — Kubernetes (manual)

Before using Helm, the deployment is done manually to understand each component.

Load the image into Minikube (required because Minikube cannot access local Docker images directly):

```bash
minikube image load flask-app:v1
```

Create the Deployment (defines what runs and how many replicas):

```bash
kubectl apply -f app/deployment.yaml
kubectl get pods
```

Create the Service (exposes the pod outside the cluster):

```bash
kubectl apply -f app/service.yaml
minikube service flask-app --url
```

Open the returned URL in your browser or run:

```bash
curl <returned-url>
# {"status": "ok"}
```

---

## Phase 3 — Helm

Install Helm:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

Create the chart scaffold:

```bash
helm create helm/flask-app
```

Helm generates a set of template files as a starting point. We keep only what we need and remove the rest:

```bash
cd helm/flask-app/templates
rm hpa.yaml httproute.yaml ingress.yaml NOTES.txt serviceaccount.yaml
rm -rf tests
rm _helpers.tpl
```

Now rewrite `deployment.yaml` and `service.yaml` using Helm variables (`{{ .Values.xxx }}`) instead of hardcoded values. This way, any configuration change only requires editing `values.yaml` — no need to touch the templates.

Update `values.yaml` with the app's configuration:

```yaml
replicaCount: 1

image:
  repository: flask-app
  tag: v1
  pullPolicy: Never

service:
  type: NodePort
  port: 80
  targetPort: 5000
```

Deploy with Helm:

```bash
helm install flask-app helm/flask-app
```

If a release with the same name already exists from the manual phase, uninstall it first:

```bash
helm uninstall flask-app
helm install flask-app helm/flask-app
```

Verify the pod is running and get the access URL:

```bash
kubectl get pods
minikube service flask-app --url
```

---

## Phase 4 — GitHub Actions

The goal is to have a fully automated GitOps pipeline using a real remote cluster.

**Workflow:**

```
git push
    ↓
GitHub Actions triggers
    ├── Build Docker image
    ├── Push to Docker Hub
    └── helm upgrade → EC2 (k3s) → updated pod
```

A **t3.small EC2 instance** running k3s acts as the Kubernetes cluster. GitHub Actions authenticates against it via a kubeconfig stored as a GitHub secret.

### 1. Provision the EC2 instance

Launch a t3.small Ubuntu 22.04 instance with the following inbound rules:
- Port 22 (SSH)
- Port 6443 (Kubernetes API)

### 2. Install k3s

The `--tls-san` flag adds the EC2 public IP to the certificate, which is required for remote access:

```bash
ssh -i k3s-key.pem ec2-user@<EC2-PUBLIC-IP>
curl -sfL https://get.k3s.io | sh -s - --tls-san <EC2-PUBLIC-IP>
sudo k3s kubectl get nodes
```

### 3. Configure the kubeconfig

Get the kubeconfig and replace the default `127.0.0.1` with the EC2 public IP:

```bash
sudo cat /etc/rancher/k3s/k3s.yaml
# Replace: server: https://127.0.0.1:6443
# With:    server: https://<EC2-PUBLIC-IP>:6443
```

### 4. Add GitHub secrets

Go to **Repository → Settings → Secrets and variables → Actions** and create:

| Secret | Value |
|--------|-------|
| `KUBECONFIG_DATA` | Contents of the kubeconfig with the public IP |
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub access token (Account Settings → Security) |

### 5. Push to main

Any push to `main` will trigger the workflow automatically. The pipeline will:

1. Check out the code
2. Log in to Docker Hub
3. Build and push the Docker image tagged with the commit SHA
4. Install Helm on the runner
5. Configure the kubeconfig from the GitHub secret
6. Run `helm upgrade --install` against the k3s cluster

Verify the deployment from the EC2:

```bash
sudo k3s kubectl get pods
# flask-app-xxxxxxxxx   1/1   Running   0   Xs
```

### Notes

- A **t2.micro** (1GB RAM) is not sufficient to run k3s and application pods simultaneously. Use **t3.small** (2GB) or larger.
- The EC2 public IP changes on every restart. Assign an **Elastic IP** to avoid having to regenerate the kubeconfig on each reboot.
- If the IP changes, the k3s TLS certificate must be regenerated: stop k3s, remove `/var/lib/rancher/k3s/server/tls`, and reinstall with the new `--tls-san`.

---

## Stack

![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?logo=kubernetes&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-0F1689?logo=helm&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?logo=githubactions&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-EC2-FF9900?logo=amazonaws)
