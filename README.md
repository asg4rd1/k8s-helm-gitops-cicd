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



## Phase 4 — GitHub Actions (coming soon)

The goal is to have a fully automated GitOps pipeline using a real remote cluster.

**Workflow:**
git push
↓
GitHub Actions triggers
├── Build Docker image
├── Push to Docker Hub
└── helm upgrade → EC2 (k3s) → updated pod

For this, a t2.micro EC2 instance running k3s acts as the Kubernetes cluster.
GitHub Actions authenticates against it via kubeconfig stored as a GitHub secret.

To achieve this we will have to follow the following steps:

first install k3s in our ec2 instace:

ssh ec2-user@IP -i k3s-key.pem
curl -sfL https://get.k3s.io | sh -

verify if installation has been done successfully
sudo kubectl get nodes

get kubeconfig to achieve github connect to our k3s cluster
sudo cat /etc/rancher/k3s/k3s.yaml

We will have to change one line before save the kubeconfig,
This line is server: https://127.0.0.1:6443
We need to change the localhost for our ec2 public IP 

When we have the kubeconfig configured, we will configure the github secrets in our repository,
These will be kubeconfig docker username and password, each in one secret.
Github - Repository settings - Secrets and variables - New secret.

Finally we have to deploy the file that we will use github actions to follow the workflow:
The steps would be:
Push to main
Start VM Ubuntu 
Download the code
Docker Hub Log in
Buildand upload the image
Connect to your EC2 and k3s cluster
Deploy with Helm


errores que han surgido:

t2 micro demasiado poco potente para correr un pod
al generar el kubeconfig se tiene que anadir un parametro para que sea valido con la ip publica


## Stack

![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?logo=kubernetes&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-0F1689?logo=helm&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?logo=githubactions&logoColor=white)