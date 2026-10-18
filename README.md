# TravelMemory — MERN Application Deployed on AWS EC2

A production-style, highly available deployment of the **TravelMemory** MERN application (MongoDB, Express, React, Node.js) on **AWS EC2**, with two load-balanced tiers, AMI-based scaling, MongoDB Atlas as the database and a custom domain.

| | |
|---|---|
| **Live app** | http://travelmemory.devopsarup.store |
| **Source application** | https://github.com/UnpredictablePrashant/TravelMemory |
| **Cloud region** | AWS `us-east-1` (N. Virginia), 2 Availability Zones |
| **Database** | MongoDB Atlas (free-tier cluster, database `travelmemory`) |

---

## Table of contents

1. [Architecture](#1-architecture)
2. [Request flow explained](#2-request-flow-explained)
3. [Deployment progress with proof](#3-deployment-progress-with-proof)
4. [Configuration used](#4-configuration-used)
5. [Security design](#5-security-design)
6. [Scalability and resilience](#6-scalability-and-resilience)
7. [Final verification](#7-final-verification)
8. [How to reproduce](#8-how-to-reproduce)
9. [Known limitations and next steps](#9-known-limitations-and-next-steps)
10. [Screenshot index](#10-screenshot-index)
11. [Repository layout](#11-repository-layout)

---

## 1. Architecture

![TravelMemory deployment architecture](Architecture/TravelMemory-Architecture.png)

> Editable source: [`Architecture/TravelMemory-Architecture.drawio`](Architecture/TravelMemory-Architecture.drawio) (open it at [draw.io](https://app.diagrams.net)).

```
Browser → Cloudflare DNS → Frontend ALB → Frontend EC2 ×2 (Nginx + React build)
   │
   └── REST API calls ──→ Backend ALB → Backend EC2 ×2 (Nginx → Node.js :3000) → MongoDB Atlas
```

| Layer | Component | Details |
|---|---|---|
| DNS | Cloudflare | `travelmemory.devopsarup.store` points to the Frontend ALB by CNAME |
| Frontend load balancer | `travelmemory-frontend-alb` | Application Load Balancer, internet-facing, HTTP:80, forwards to `travelmemory-frontend-tg` |
| Frontend servers | `frontend-master` (us-east-1a), `frontend-2` (us-east-1b) | `t3.small`, Nginx serving the static React build |
| Backend load balancer | `travelmemory-backend-alb` | Application Load Balancer, internet-facing, HTTP:80, forwards to `travelmemory-backend-tg` |
| Backend servers | `backend-master` (us-east-1a), `backend-2` (us-east-1b) | `t2.micro`, Nginx reverse proxy to Node.js on port 3000, run by systemd |
| Database | MongoDB Atlas | Managed cluster, connection string stored in the backend `.env` |

---

## 2. Request flow explained

1. A visitor opens `travelmemory.devopsarup.store`. **Cloudflare DNS** resolves it to the **Frontend ALB** (a CNAME, because an ALB has no fixed IP).
2. The **Frontend ALB** distributes requests across `frontend-master` and `frontend-2`, which are in different Availability Zones. **Nginx** serves the React build from `/var/www/html`, and the `try_files` rule makes client-side routes work on refresh.
3. The React app runs **in the visitor's browser**. Its `baseUrl` (in `frontend/src/url.js`) points to the **Backend ALB**, so API calls go from the browser straight to the backend tier. This is the purple dashed line in the diagram.
4. The **Backend ALB** distributes API calls across `backend-master` and `backend-2`. On each server **Nginx** proxies port 80 to the **Node.js/Express** app on port 3000.
5. The backend reads and writes trip data in **MongoDB Atlas** over a TLS `mongodb+srv` connection.

Each server tier is built once as a **master** instance, imaged as an **AMI**, then launched a second time in the other Availability Zone. Node.js runs under **systemd**, so every instance launched from the AMI starts serving on its own.

---

## 3. Deployment progress with proof

Each step below links to the screenshot that proves it. All images are in the [`Screenshots/`](Screenshots) folder.

### Step 1: EC2 servers

Four Ubuntu 26.04 instances, two per tier, spread over two Availability Zones. All are `Running` with their status checks passed. The frontend servers are `t3.small` because the React build needs more memory than a `t2.micro` offers.

| Instance | Type | AZ | Security group |
|---|---|---|---|
| `backend-master` | t2.micro | us-east-1a | `backend-sg` |
| `backend-2` | t2.micro | us-east-1b | `backend-sg` |
| `frontend-master` | t3.small | us-east-1a | `frontend-sg` |
| `frontend-2` | t3.small | us-east-1b | `frontend-sg` |

![Backend and frontend EC2 instances](Screenshots/Back-Front-Ubuntu-Servers.png)

*Proof: [Screenshots/Back-Front-Ubuntu-Servers.png](Screenshots/Back-Front-Ubuntu-Servers.png)*

### Step 2: Backend Nginx reverse proxy (backend-master)

The default Nginx site on `backend-master` was changed so that `location /` proxies to the Node.js app on `localhost:3000` and forwards the client's `Host`, `X-Real-IP` and `X-Forwarded-For` headers.

![Backend Nginx configuration](Screenshots/backend-nginx-config.png)

*Proof: [Screenshots/backend-nginx-config.png](Screenshots/backend-nginx-config.png)*

### Step 3: Frontend Nginx configuration (frontend-master)

On `frontend-master` Nginx serves the React build from `/var/www/html`. The `location /` block uses `try_files $uri /index.html;` so that refreshing or deep-linking to a React Router page does not return a 404.

![Frontend Nginx configuration](Screenshots/Frontend-nginx-config.png)

*Proof: [Screenshots/Frontend-nginx-config.png](Screenshots/Frontend-nginx-config.png)*

### Step 4: Security groups

Instances accept web traffic **only from the load balancer's security group**, and SSH only from a single admin IP.

**Backend security group** (`backend-sg`):

![Backend security group](Screenshots/Backend-Security-Group.png)

*Proof: [Screenshots/Backend-Security-Group.png](Screenshots/Backend-Security-Group.png)*

**Frontend security group** (`frontend-sg`):

![Frontend security group](Screenshots/Frontend-Security-Group.png)

*Proof: [Screenshots/Frontend-Security-Group.png](Screenshots/Frontend-Security-Group.png)*

### Step 5: Machine images (AMIs)

After the master servers were verified working, each was imaged. The AMIs were then used to launch `backend-2` and `frontend-2` in the second Availability Zone.

| AMI | Status |
|---|---|
| `travelmemory-backend-ami` | Available |
| `travelmemory-frontend-ami` | Available |

![Backend and frontend AMIs](Screenshots/Back-Front-AMIs.png)

*Proof: [Screenshots/Back-Front-AMIs.png](Screenshots/Back-Front-AMIs.png)*

### Step 6: Target groups

Each tier has its own target group (target type *Instance*, protocol HTTP, port 80) with both servers registered. Both show **2 / 2 healthy**, one target in each Availability Zone.

**Backend target group** (`travelmemory-backend-tg`):

![Backend target group](Screenshots/Backend-Target-Group.png)

*Proof: [Screenshots/Backend-Target-Group.png](Screenshots/Backend-Target-Group.png)*

**Frontend target group** (`travelmemory-frontend-tg`):

![Frontend target group](Screenshots/Frontend-Target-Group.png)

*Proof: [Screenshots/Frontend-Target-Group.png](Screenshots/Frontend-Target-Group.png)*

### Step 7: Application Load Balancers

Two internet-facing Application Load Balancers, both `Active`, each attached to subnets in `us-east-1a` and `us-east-1b`, with an HTTP:80 listener forwarding 100% of traffic to the matching target group.

**Backend ALB** (`travelmemory-backend-alb`):

![Backend ALB](Screenshots/Backend-ALB.png)

*Proof: [Screenshots/Backend-ALB.png](Screenshots/Backend-ALB.png)*

**Frontend ALB** (`travelmemory-frontend-alb`):

![Frontend ALB](Screenshots/Frontend-ALB.png)

*Proof: [Screenshots/Frontend-ALB.png](Screenshots/Frontend-ALB.png)*

### Step 8: Custom domain through Cloudflare

The custom domain is managed in Cloudflare. A **CNAME** record maps `travelmemory` (the app hostname) to the Frontend ALB's DNS name. A CNAME is used instead of an A record because an ALB has no fixed IP address, and an A record to a single EC2 instance would bypass load balancing.

| Type | Name | Target |
|---|---|---|
| CNAME | `travelmemory` | Frontend ALB DNS name (`travelmemory-frontend-alb-….us-east-1.elb.amazonaws.com`) |

<!-- Optional: add a Cloudflare DNS screenshot as Screenshots/Cloudflare-DNS.png and link it here. -->

### Step 9: Application live

The application loads at the custom domain and displays trip records stored in MongoDB Atlas. The data travels from the browser through the Backend ALB, a backend EC2 server and then Atlas, which proves the whole chain works end to end.

![TravelMemory running on the custom domain](Screenshots/TravelMemoryPage.png)

*Proof: [Screenshots/TravelMemoryPage.png](Screenshots/TravelMemoryPage.png)*

---

## 4. Configuration used

### Backend environment (`backend/.env`)

The real connection string is **never committed** to this repository.

```env
MONGO_URI=mongodb+srv://arupchakraborty7-db-user:**************@arupcluster.y3kqmyo.mongodb.net/travelmemory
PORT=3000
```

### Backend systemd service

Running Node.js under systemd restarts it on crash and starts it on every boot, which is what lets AMI clones come up serving traffic by themselves.

```ini
[Unit]
Description=TravelMemory Backend
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/TravelMemory/backend
EnvironmentFile=/home/ubuntu/TravelMemory/backend/.env
ExecStart=/usr/bin/node index.js
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Backend Nginx (reverse proxy) — see [screenshot](Screenshots/backend-nginx-config.png)

```nginx
location / {
    proxy_pass http://localhost:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

### Frontend Nginx (static site) — see [screenshot](Screenshots/Frontend-nginx-config.png)

```nginx
root /var/www/html;
index index.html index.htm;

location / {
    try_files $uri /index.html;
}
```

### Frontend to backend connection (`frontend/src/url.js`)

```js
export const baseUrl = "http://travelmemory-backend-alb-2063493088.us-east-1.elb.amazonaws.com";
```

This change is made on **both** frontend servers after the Backend ALB exists, followed by `npm run build` and copying the build to `/var/www/html`.

---

## 5. Security design

| Control | Implementation | Proof |
|---|---|---|
| No direct web access to servers | `frontend-sg` and `backend-sg` allow HTTP 80 **only from the ALB security group** | [Frontend SG](Screenshots/Frontend-Security-Group.png), [Backend SG](Screenshots/Backend-Security-Group.png) |
| Restricted SSH | Port 22 allowed only from a single admin IP (`/32`) | Same screenshots |
| Secrets out of source control | Database credentials only in `backend/.env` on the servers | Section 4 |
| Managed database | MongoDB Atlas with a dedicated DB user and TLS (`mongodb+srv`) | Section 1 |
| Separate tiers | Frontend and backend have independent security groups, load balancers and target groups | Steps 4 to 7 above |

---

## 6. Scalability and resilience

- **Two Availability Zones:** every tier has one server in `us-east-1a` and one in `us-east-1b`, so the loss of a zone does not take the app down.
- **Load balancing:** each ALB spreads traffic across both servers in its tier ([Frontend ALB](Screenshots/Frontend-ALB.png), [Backend ALB](Screenshots/Backend-ALB.png)).
- **Health checks:** target groups report **2 / 2 healthy** ([Backend TG](Screenshots/Backend-Target-Group.png), [Frontend TG](Screenshots/Frontend-Target-Group.png)). An unhealthy server is taken out of rotation automatically.
- **Self-healing processes:** systemd `Restart=always` for Node.js, and Nginx enabled at boot.
- **Repeatable scaling:** AMIs ([screenshot](Screenshots/Back-Front-AMIs.png)) let a new server be launched in minutes with the application already installed.
- **Independent tiers:** the frontend and backend scale separately.

---

## 7. Final verification

- [x] The site loads at the custom domain ([screenshot](Screenshots/TravelMemoryPage.png))
- [x] Trip records load from MongoDB Atlas through the Backend ALB
- [x] Both frontend targets and both backend targets show healthy
- [x] Both load balancers are `Active` and internet-facing
- [x] Security groups limit web traffic to the load balancers

---

## 8. How to reproduce

Condensed command reference (Ubuntu 26.04, Node.js 22.22.1):

```bash
# --- Backend server ---
sudo apt update
sudo apt install -y nodejs git nginx
git clone https://github.com/UnpredictablePrashant/TravelMemory.git
cd TravelMemory/backend && npm install
nano .env                                  # MONGO_URI and PORT=3000
# create /etc/systemd/system/travelmemory-backend.service (see section 4)
sudo systemctl daemon-reload
sudo systemctl enable --now travelmemory-backend
# configure Nginx proxy_pass to localhost:3000, then:
sudo nginx -t && sudo systemctl restart nginx

# --- Frontend server ---
cd TravelMemory/frontend
nano src/url.js                            # baseUrl = Backend ALB DNS name
npm install && npm run build
sudo rm -rf /var/www/html/*
sudo cp -r build/* /var/www/html/
# set try_files $uri /index.html; in Nginx, then:
sudo nginx -t && sudo systemctl restart nginx
```

AWS order of operations: Atlas cluster → master EC2 instances → verify → AMIs → second instance per tier from each AMI (other AZ) → target groups → ALBs → point the frontend at the Backend ALB and rebuild → Cloudflare CNAME.

---

## 9. Known limitations and next steps

- **HTTP only:** the site is served over plain HTTP (the browser shows "Not secure"). Next step: an AWS Certificate Manager certificate, an HTTPS:443 listener on both ALBs and Cloudflare SSL mode *Full (strict)*.
- **Atlas network access:** open to `0.0.0.0/0` for simplicity. A tighter setup would use private subnets with a NAT gateway and allow-list only its Elastic IP.
- **Manual scaling:** two fixed servers per tier. An **Auto Scaling Group** with a launch template would scale automatically.
- **Public IPs change** when an instance is stopped and started. The app is unaffected because all traffic goes through the load balancers.

---

## 10. Screenshot index

| # | Screenshot | What it proves |
|---|---|---|
| 1 | [`Back-Front-Ubuntu-Servers.png`](Screenshots/Back-Front-Ubuntu-Servers.png) | Four EC2 instances running across two AZs |
| 2 | [`backend-nginx-config.png`](Screenshots/backend-nginx-config.png) | Nginx reverse proxy to Node.js on port 3000 |
| 3 | [`Frontend-nginx-config.png`](Screenshots/Frontend-nginx-config.png) | Nginx serving the React build with `try_files` |
| 4 | [`Backend-Security-Group.png`](Screenshots/Backend-Security-Group.png) | Backend firewall rules |
| 5 | [`Frontend-Security-Group.png`](Screenshots/Frontend-Security-Group.png) | Frontend firewall rules |
| 6 | [`Back-Front-AMIs.png`](Screenshots/Back-Front-AMIs.png) | Backend and frontend AMIs available |
| 7 | [`Backend-Target-Group.png`](Screenshots/Backend-Target-Group.png) | Backend target group, 2 / 2 healthy |
| 8 | [`Frontend-Target-Group.png`](Screenshots/Frontend-Target-Group.png) | Frontend target group, 2 / 2 healthy |
| 9 | [`Backend-ALB.png`](Screenshots/Backend-ALB.png) | Backend load balancer active and forwarding |
| 10 | [`Frontend-ALB.png`](Screenshots/Frontend-ALB.png) | Frontend load balancer active and forwarding |
| 11 | [`TravelMemoryPage.png`](Screenshots/TravelMemoryPage.png) | Live application on the custom domain |

---

## 11. Repository layout

```
.
├── README.md
├── Architecture/
│   ├── TravelMemory-Architecture.png
│   └── TravelMemory-Architecture.drawio
├── Screenshots/
│   ├── Back-Front-Ubuntu-Servers.png
│   ├── backend-nginx-config.png
│   ├── Frontend-nginx-config.png
│   ├── Backend-Security-Group.png
│   ├── Frontend-Security-Group.png
│   ├── Back-Front-AMIs.png
│   ├── Backend-Target-Group.png
│   ├── Frontend-Target-Group.png
│   ├── Backend-ALB.png
│   ├── Frontend-ALB.png
│   └── TravelMemoryPage.png
├── backend/
└── frontend/
```
