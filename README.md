![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-47A248?style=for-the-badge&logo=mongodb&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-009639?style=for-the-badge&logo=nginx&logoColor=white)
![NodeJS](https://img.shields.io/badge/Node.js-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)
![React](https://img.shields.io/badge/React-20232A?style=for-the-badge&logo=react&logoColor=61DAFB)

# 🚀 Full Stack Real-Time Chat App — Kubernetes Deployment

Welcome to the **Full Stack Realtime Chat App** — a scalable and secure real-time chat experience deployed on **Kubernetes** with Ingress Controller, Persistent Storage, and JWT Authentication.

---

## 📌 Table of Contents

- [Introduction](#introduction)
- [Features](#features)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Kubernetes Deployment](#kubernetes-deployment)
- [Kubernetes Files](#kubernetes-files)
- [Problems I Solved](#problems-i-solved)
- [Getting Started](#getting-started)
- [Deploy on Kubernetes](#deploy-on-kubernetes)
- [Project Snapshots](#project-snapshots)
- [Future Plans](#future-plans)
- [License](#license)

---

## 📝 Introduction

This project provides a real-time chat experience that is both scalable and secure. I took this full stack application and deployed it completely on **Kubernetes from scratch** — writing all manifest files, configuring Ingress Controller, fixing JWT cookie authentication issues, and setting up Persistent Storage for MongoDB.

---

## ✨ Features

- **Real-time Messaging** — Send and receive messages instantly using Socket.io
- **User Authentication & Authorization** — Securely manage user access with JWT
- **Scalable & Secure Architecture** — Built to handle large volumes of traffic and data
- **Modern UI Design** — A user-friendly interface crafted with React and TailwindCSS
- **Profile Management** — Users can upload and update their profile pictures
- **Online Status** — View real-time online/offline status of users

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React, TailwindCSS, DaisyUI |
| Backend | Node.js, Express, Socket.io |
| Database | MongoDB |
| Authentication | JWT (JSON Web Tokens) |
| State Management | Zustand |
| Containerization | Docker |
| Orchestration | Kubernetes |
| Web Server | Nginx |
| Ingress | Nginx Ingress Controller |
| Storage | Kubernetes PersistentVolume |

---

## 🏗️ Architecture

```
                   ┌─────────────────────────────────────────┐
                   │          Kubernetes Cluster              │
                   │                                          │
                   │   ┌──────────────────────────────────┐  │
  User Browser ─────────► Ingress Controller (Nginx)       │  │
  http://ip:31276  │   └───────┬──────────────┬───────────┘  │
                   │           │              │               │
                   │      path:/         path:/api            │
                   │                    path:/socket.io       │
                   │           │              │               │
                   │   ┌───────▼──────┐ ┌────▼───────────┐  │
                   │   │   Frontend   │ │    Backend     │  │
                   │   │   React +    │ │   Node.js +    │  │
                   │   │   Nginx      │ │   Express +    │  │
                   │   │   Port: 80   │ │   Socket.io    │  │
                   │   └──────────────┘ │   Port: 5001   │  │
                   │                    └────────┬───────┘  │
                   │                             │           │
                   │                    ┌────────▼───────┐  │
                   │                    │    MongoDB     │  │
                   │                    │   Port: 27017  │  │
                   │                    │  PersistentVol │  │
                   │                    └────────────────┘  │
                   └─────────────────────────────────────────┘
```

---

## ☸️ Kubernetes Deployment

```
Namespace: chatapp
│
├── Deployments
│   ├── frontend-deployment    (React + Nginx + ConfigMap)
│   ├── backend-deployment     (Node.js + Express + Socket.io)
│   └── mongodb-deployment     (MongoDB with credentials)
│
├── Services
│   ├── frontend-service       (NodePort  → 30007)
│   ├── backend                (ClusterIP → 5001)
│   └── mongodb                (ClusterIP → 27017)
│
├── ConfigMap
│   └── nginx-config           (Custom Nginx with Cookie forwarding)
│
├── Ingress
│   └── chatapp-ingress        (Routes / → frontend, /api → backend)
│
└── Storage
    ├── mongo-pv               (PersistentVolume  → /mnt/data/mongodb)
    └── mongo-pvc              (PersistentVolumeClaim → 1Gi)
```

---

## 📁 Kubernetes Files

| File | Purpose |
|---|---|
| `namespace.yml` | Creates `chatapp` namespace |
| `backend-deployment.yml` | Deploys backend with env vars (JWT, MongoDB URI) |
| `frontend-deployment.yml` | Deploys frontend with Nginx ConfigMap mounted |
| `mongodb-deployment.yml` | Deploys MongoDB with root credentials |
| `backend-service.yml` | Exposes backend internally on port 5001 |
| `frontend-service.yml` | Exposes frontend externally on NodePort 30007 |
| `mongodb-service.yml` | Exposes MongoDB internally on port 27017 |
| `mongo-pv.yml` | Creates PersistentVolume at `/mnt/data/mongodb` |
| `mongo-pvc.yml` | Creates PersistentVolumeClaim for MongoDB data |
| `nginx-configmap.yml` | Custom Nginx config with Cookie forwarding fix |
| `chatapp-ingress.yml` | Ingress routing rules for all services |

---

## 🔧 Problems I Solved

### ❌ Problem 1 — Unauthorized: No Token Provided

**What was happening:**
```
User logs in
→ Backend creates JWT token and sets it as a Cookie
→ Cookie sent back to browser through Nginx
→ Browser sends cookie in next request
→ Nginx strips the cookie (not forwarding it)
→ Backend receives request with NO cookie
→ Returns "Unauthorized - No Token Provided" ❌
```

**Root Cause:**
Nginx config was missing the cookie forwarding header.

**Fix — Added to Nginx ConfigMap:**
```nginx
location /api/ {
    proxy_pass http://backend:5001/api/;
    proxy_set_header Cookie $http_cookie;  ← This line was missing!
}
```

---

### ❌ Problem 2 — Secure Cookie Dropped Over HTTP

**What was happening:**
```
Backend sets JWT cookie with secure: true
→ App running on HTTP (not HTTPS)
→ Browser silently drops secure cookies over HTTP
→ Next request has no cookie
→ "Unauthorized - No Token Provided" ❌
```

**Root Cause:**
In `utils.js`:
```javascript
secure: process.env.NODE_ENV !== "development"
// NODE_ENV=production → secure=true → cookie dropped over HTTP!
```

**Fix — Changed in `backend-deployment.yml`:**
```yaml
- name: NODE_ENV
  value: "development"   # secure: false → cookie works over HTTP ✅
```

---

### ❌ Problem 3 — Ingress rewrite-target Breaking API Routes

**What was happening:**
```
Browser calls → /api/auth/login
Ingress rewrites → /
Backend receives → / instead of /api/auth/login
Returns → Cannot GET / ❌
```

**Root Cause:**
```yaml
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /  # Breaking all /api paths!
```

**Fix:**
Removed the `rewrite-target` annotation completely from Ingress manifest.

---

## 🚀 Getting Started

### Prerequisites

- **[Node.js](https://nodejs.org/)** (v14 or higher)
- **[Docker](https://www.docker.com/get-started)** (for containerizing the app)
- **[kubectl](https://kubernetes.io/docs/tasks/tools/)** (for Kubernetes)
- **[Git](https://git-scm.com/downloads)** (to clone the repository)

### Environment Configuration

Create a `.env` file in the root directory:

```env
# Database Configuration
MONGODB_URI=mongodb://root:admin@mongo:27017/chatApp?authSource=admin&retryWrites=true&w=majority

# JWT Configuration
JWT_SECRET=your_jwt_secret_key

# Server Configuration
PORT=5001
NODE_ENV=development
```

### Run with Docker Compose

```bash
git clone https://github.com/AhmadButt97/Full-Stack-Chatapp.git
cd Full-Stack-Chatapp
docker-compose up -d --build
```

Access the app at: `http://localhost`

---

## ☸️ Deploy on Kubernetes

### Step 1 — Clone the repo
```bash
git clone https://github.com/AhmadButt97/Full-Stack-Chatapp.git
cd Full-Stack-Chatapp
```

### Step 2 — Create namespace
```bash
kubectl apply -f k8s/namespace.yml
```

### Step 3 — Apply storage
```bash
kubectl apply -f k8s/mongo-pv.yml
kubectl apply -f k8s/mongo-pvc.yml
```

### Step 4 — Deploy everything
```bash
kubectl apply -f k8s/
```

### Step 5 — Install Ingress Controller
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/baremetal/deploy.yaml
```

### Step 6 — Verify everything is running
```bash
kubectl get all -n chatapp
kubectl get ingress -n chatapp
```

### Step 7 — Access the app
```
# Via NodePort
http://<your-node-ip>:30007

# Via Ingress Controller
http://<your-node-ip>:31276
```

---

## 📚 Project Snapshots

![Settings](frontend/public/settings.png)

![Chat](frontend/public/chat.png)

![Logout](frontend/public/logout.png)

![Login](frontend/public/login.png)

---

## 🧠 What I Learned

| Topic | What I Learned |
|---|---|
| JWT Cookies in K8s | How cookies travel between browser → Nginx → Backend |
| Nginx ConfigMap | How to override files inside pods without rebuilding image |
| Secure Cookies | Why secure cookies fail over HTTP and how to fix it |
| Ingress Controller | How to route traffic using path-based routing |
| PersistentVolume | How to persist MongoDB data across pod restarts |
| K8s Debugging | Using `kubectl logs`, `kubectl exec`, `kubectl describe` |

---

## 🔮 Future Plans

- [ ] **CI/CD Pipelines** — Jenkins pipeline to auto build and deploy on every git push
- [ ] **Cloud Deployment** — Deploy on AWS EKS or GCP GKE
- [ ] **HTTPS/SSL** — Add SSL certificate using Cert-Manager
- [ ] **Monitoring** — Add Prometheus and Grafana for pod monitoring
- [ ] **Helm Chart** — Package all K8s manifests into a Helm chart
- [ ] **Feature Expansion** — Group chats, media sharing, user status updates

---

## 👨‍💻 Author

**Ahmad Butt**
- GitHub: [@AhmadButt97](https://github.com/AhmadButt97)
