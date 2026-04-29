# Docker Lab: Containerizing a Three-Tier Application
**INET 4031 - Introductions to Systems**

This lab introduces Docker and Docker Compose by having you containerize a
real, multi-service application. You will package three components: Apache,
Flask, and MariaDB. These will be packaged into separate containers and wired together so they function as a complete application.

The application code and scaffolding are provided. Your job is to complete the Dockerfiles, verify the stack runs correctly, and document your work below.

> **Directions and explanations for this lab are on the repository Wiki.**
> Refer to the Wiki pages for step-by-step instructions.

---

*The sections below are for you to fill out. Replace each placeholder with your own content before submitting. Having a detailed README is the best practice for showing your work in future GitHub repositories.*

---

# Project Overview

This application is a three-tier web app built around a simple ticketing system (`ticketdb`). Users interact with a web interface served by **Apache**, which forwards requests to a **Flask** backend. Flask handles the application logic and reads/writes data to a **MariaDB** database.

Each tier runs in its own Docker container, and Docker Compose wires them together over a private bridge network. The goal of this lab is to write the Dockerfiles that build the Apache and Flask containers, configure the environment, and confirm the full stack runs correctly end-to-end.

```
Browser → Apache (port 80) → Flask (port 5000) → MariaDB (port 3306)
```

# Prerequisites

The following must be installed and available on your VM before starting:

- **Docker** (v20.10 or later) — used to build and run containers
- **Docker Compose** (v2 or later) — used to orchestrate the multi-container stack
- **Git** — to clone this repository

You can verify both are present with:

```bash
docker --version
docker compose version
```

# Getting Started

Follow these steps from a fresh clone to get the stack running:

```bash
# 1. Clone the repository
git clone https://github.com/fenkur/inet4031-testlab12.git
cd inet4031-testlab12

# 2. Create your environment file from the template
cp .env.example .env

# 3. Open .env and set your own passwords (see Configuration below)
nano .env

# 4. Build all container images and start the stack
docker compose up --build

# 5. Open a browser and navigate to:
http://localhost
```

To stop the stack, press `Ctrl+C` or run:

```bash
docker compose down
```

# Configuration

Environment variables are stored in a `.env` file that you create locally from the provided template. This file holds database credentials and is **not** committed to the repository (it is listed in `.gitignore`).

To set it up:

```bash
cp .env.example .env
```

Then edit `.env` and replace the placeholder values:

| Variable | Description | Example |
|---|---|---|
| `DB_ROOT_PASSWORD` | MariaDB root account password | `s3cur3rootpw!` |
| `DB_NAME` | Name of the database Flask connects to | `ticketdb` |
| `DB_USER` | Database user for the Flask app | `flaskuser` |
| `DB_PASSWORD` | Password for the Flask app's database user | `s3cur3flaskpw!` |

> **Never commit your `.env` file.** It contains credentials and should stay on your local machine only.

# Verification

Once the stack is up, run the provided check script to confirm all three services are healthy and communicating:

```bash
bash check-lab.sh
```

A passing run will show each service — Apache, Flask, and MariaDB — successfully responding. If any check fails, review the logs for that container:

```bash
docker compose logs db     # MariaDB logs
docker compose logs app    # Flask logs
docker compose logs web    # Apache logs
```

You can also manually verify by visiting `http://localhost` in a browser and confirming the application loads without errors.

---

# Lab 13: Migrating to Kubernetes

In Lab 13, the same three-tier application was migrated from Docker Compose to Kubernetes. Instead of running everything on a single host with a Compose file, each service is now managed as a Kubernetes Deployment running inside a k3s cluster. Kubernetes handles scheduling, self-healing, and service discovery automatically.

```
Browser → Apache NodePort (port 30080) → Flask Service (port 5000) → MariaDB Service (port 3306)
```

## What Changed

- Docker Compose is no longer used to run the application
- Each service (Apache, Flask, MariaDB) is defined as a Kubernetes Deployment with a corresponding Service
- Credentials are stored in a Kubernetes Secret (`db-credentials`) instead of a `.env` file
- MariaDB data is persisted using a PersistentVolumeClaim (`db-pvc`)
- The application is accessible via a NodePort Service on port `30080` instead of port `80`

## Deploying the Application

```bash
# 1. Create the namespace
kubectl create namespace ticket-app

# 2. Create the credentials secret
bash create-secret.sh

# 3. Apply all Kubernetes manifests
kubectl apply -f k8s/

# 4. Verify all pods are running
kubectl get pods -n ticket-app
```

## Accessing the Dashboard

Once all pods show `Running`, open a browser and navigate to:

```
http://<VM-IP>:30080
```

Replace `<VM-IP>` with the IP address of your VM. You can find it with:

```bash
hostname -I
```

## Verification

Run the provided check script to confirm all services are healthy:

```bash
bash check-lab13.sh
```

If any checks fail, inspect the logs for the relevant deployment:

```bash
kubectl logs -n ticket-app deployment/app   # Flask logs
kubectl logs -n ticket-app deployment/db    # MariaDB logs
kubectl logs -n ticket-app deployment/web   # Apache logs
```

## Feedback

One thing that tripped me up was that Kubernetes does not automatically restart pods when a Secret is updated — you have to manually rollout restart the affected deployment. It would be helpful to have a note about this in the lab instructions, since the symptom (access denied) looks like a credentials mismatch rather than a stale pod issue.