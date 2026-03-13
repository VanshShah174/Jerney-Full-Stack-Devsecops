# Jerney — Full-Stack DevSecOps Blog Platform

A production-grade, 3-tier blog platform built with React, Node.js, and PostgreSQL — deployed on AWS using Docker, Kubernetes (EKS Auto Mode), Terraform, and a fully automated GitHub Actions DevSecOps pipeline.

![React](https://img.shields.io/badge/React-18-61DAFB?style=flat-square&logo=react)
![Node.js](https://img.shields.io/badge/Node.js-20-339933?style=flat-square&logo=node.js)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?style=flat-square&logo=postgresql)
![Docker](https://img.shields.io/badge/Docker-Containerized-2496ED?style=flat-square&logo=docker)
![Kubernetes](https://img.shields.io/badge/Kubernetes-EKS_Auto_Mode-326CE5?style=flat-square&logo=kubernetes)
![Terraform](https://img.shields.io/badge/Terraform-IaC-7B42BC?style=flat-square&logo=terraform)
![GitHub Actions](https://img.shields.io/badge/CI/CD-GitHub_Actions-2088FF?style=flat-square&logo=githubactions)

---

## Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Frontend   │────▶│   Backend    │────▶│  PostgreSQL  │
│  React + Vite│◀────│ Node.js +    │◀────│   (EBS PVC   │
│  Nginx:8080  │     │ Express:5000 │     │   on EKS)    │
└──────────────┘     └──────────────┘     └──────────────┘
        │                    │
        └────────────────────┘
               AWS EKS (Auto Mode)
               VPC + Private Subnets
               NAT Gateway
```

---

## Branch Strategy

| Branch | Purpose |
|--------|---------|
| `main` | Application source code + EC2 bare-metal deployment |
| `devops` | Full DevSecOps — Docker, Kubernetes (EKS), Terraform, CI/CD, security scanning |

> Switch to `devops` branch for the full infrastructure implementation:
> ```bash
> git checkout devops
> ```

---

## DevSecOps CI/CD Pipeline

Every push triggers a 7-stage automated pipeline via GitHub Actions:

```
Push / PR
   │
   ├── 1. Lint          — ESLint on backend + frontend
   ├── 2. SCA           — npm audit (dependency vulnerability scan)
   ├── 3. Build & Push  — Docker images → GitHub Container Registry (GHCR)
   ├── 4. Image Scan    — Trivy (CVE scan: CRITICAL + HIGH)
   ├── 5. IaC Scan      — Checkov on Terraform + Kubernetes manifests
   ├── 6. Dockerfile    — Hadolint (Dockerfile best practices)
   └── 7. Manifest      — Auto-update K8s image tags on push to main
```

---

## Project Structure

```
Jerney-devops/
├── .github/
│   └── workflows/
│       └── ci-cd.yml          # 7-stage DevSecOps pipeline
├── frontend/                  # React 18 + Vite
│   ├── src/                   # Components, pages, routing
│   ├── nginx.conf             # Nginx config (non-root, port 8080)
│   └── Dockerfile
├── backend/                   # Node.js + Express REST API
│   ├── src/                   # Routes, DB connection (pg)
│   └── Dockerfile
├── k8s/
│   └── jerney.yaml            # Full K8s manifest — Namespace, Secrets,
│                              # PVC (EBS gp3), Deployments, Services,
│                              # NetworkPolicies
├── terraform/
│   ├── main.tf                # EKS Auto Mode + VPC (3 AZs, NAT Gateway)
│   ├── variables.tf
│   ├── outputs.tf
│   ├── provider.tf
│   └── terraform.tfvars
├── deploy/
│   ├── setup.sh               # One-click EC2 setup script
│   └── jerney-nginx.conf      # Nginx reverse proxy config
└── docker-compose.yml         # Local development stack
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 18, Vite, React Router v6, Axios, Nginx |
| Backend | Node.js 20, Express, pg (PostgreSQL client) |
| Database | PostgreSQL 16, EBS gp3 PersistentVolume |
| Containers | Docker, Docker Compose, GHCR |
| Orchestration | Kubernetes, AWS EKS Auto Mode, Karpenter |
| Infrastructure | Terraform, AWS VPC, Private/Public Subnets |
| CI/CD | GitHub Actions |
| Security | Trivy, Checkov, Hadolint, npm audit |

---

## Security Highlights

- Containers run as **non-root** users (UID 1000 / 101)
- **Read-only root filesystems** on all containers
- **NetworkPolicies** — DB only accepts traffic from backend; backend only from frontend
- **Secrets encrypted at rest** in EKS via envelope encryption (KMS)
- EKS **audit logging** enabled (api, audit, authenticator, controllerManager, scheduler)
- `no-new-privileges` security opt on all containers
- Trivy scans block on **CRITICAL/HIGH** CVEs
- Checkov enforces IaC security best practices

---

## Running Locally

### Option 1 — Docker Compose (Recommended)

Requires: Docker Desktop

```bash
git clone https://github.com/VanshShah174/Jerney-devops.git
cd Jerney-devops
docker compose up --build
```

| Service | URL |
|---------|-----|
| Frontend | http://localhost |
| Backend API | http://localhost (proxied via Nginx) |
| PostgreSQL | localhost:5432 |

```bash
# Stop all services
docker compose down
```

### Option 2 — Local Dev (No Docker)

Requires: Node.js 20+, PostgreSQL 16+

```bash
# Terminal 1 — Backend
cd backend
npm install
export DB_HOST=localhost DB_PORT=5432 DB_USER=jerney_user \
       DB_PASSWORD=jerney_pass_2026 DB_NAME=jerney_db PORT=5000
npm start

# Terminal 2 — Frontend
cd frontend
npm install
npm run dev
# http://localhost:3000
```

---

## Deploy on AWS EC2 (main branch)

### Prerequisites
- EC2 instance: Ubuntu 22.04+
- Security Group: ports **22** (SSH) and **80** (HTTP) open

```bash
# 1. Upload project to EC2
scp -r -i your-key.pem ./Jerney-devops ubuntu@<EC2_PUBLIC_IP>:~/Jerney

# 2. SSH in
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>

# 3. Run one-click setup (installs Node.js, PostgreSQL, Nginx, PM2)
cd ~/Jerney
chmod +x deploy/setup.sh
./deploy/setup.sh

# 4. Visit
open http://<EC2_PUBLIC_IP>
```

**Useful commands on EC2:**
```bash
pm2 status                          # Backend process status
pm2 logs                            # Backend logs
pm2 restart all                     # Restart backend
sudo systemctl restart nginx        # Restart Nginx
sudo -u postgres psql -d jerney_db  # Connect to DB
```

---

## Deploy on AWS EKS (devops branch)

```bash
git checkout devops

# 1. Provision infrastructure
cd terraform
terraform init
terraform plan
terraform apply

# 2. Configure kubectl
aws eks update-kubeconfig --name <cluster-name> --region <region>

# 3. Deploy to Kubernetes
kubectl apply -f k8s/jerney.yaml

# 4. Access the app
kubectl port-forward svc/jerney-frontend 8080:80 -n jerney
# http://localhost:8080
```

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/health` | Health check |
| GET | `/api/posts` | Get all posts |
| GET | `/api/posts/:id` | Get single post with comments |
| POST | `/api/posts` | Create a post |
| PUT | `/api/posts/:id` | Update a post |
| DELETE | `/api/posts/:id` | Delete a post |
| GET | `/api/comments/post/:postId` | Get comments for a post |
| POST | `/api/comments` | Create a comment |
| DELETE | `/api/comments/:id` | Delete a comment |
