# Multi-client Docker Swarm Deployment on AWS EC2

## Overview

This project demonstrates a **Docker Swarm deployment** of multiple microservices with Nginx, Node.js, Python (FastAPI), Prometheus, Grafana, and cAdvisor.  
The setup simulates multiple clients accessing different services via **Nginx virtual hosts**.

---

## **Project Structure**
.
├── docker-compose.yml # Swarm stack configuration
├── nginx.conf # Nginx configuration for virtual hosts
├── prometheus.yml # Prometheus scrape configuration
├── Node-app/ # Node.js (NestJS) application
├── Python-app/ # Python (FastAPI) application
├── README.md # This file


---
# 4️⃣ Scaling & Deployment Scenario
Docker Swarm allows services to be **scaled and updated** without downtime. Below are examples using the `myapp_node-app` service.
---

### **Manual Scale Command**
Scale the service to 5 replicas:

```bash
docker service scale myapp_node-app=5
Verify the scale:
docker service ls
docker service ps myapp_node-app
Swarm will automatically create new tasks (containers) to reach the desired replica count.

Rolling Update
Update the service image with zero downtime:

docker service update \
  --image maheshkhatana1210/node-app:v2 \
  --update-parallelism 1 \
  --update-delay 10s \
  myapp_node-app

--update-parallelism 1: Update one replica at a time
--update-delay 10s: Wait 10 seconds between updating replicas

Verify update:
docker service ps myapp_node-app
Swarm will stop old tasks only after new tasks are healthy, ensuring zero-downtime deployment.

Zero-Downtime Deployment
Swarm ensures that at least the minimum number of replicas are running during updates.
Requests are automatically routed to healthy containers through internal load balancing.

How Swarm Ensures Availability
Swarm monitors the desired state and automatically restarts failed containers.
If a node goes down, Swarm reschedules tasks to other available nodes.
Health checks are used to ensure containers are ready before routing traffic.

Service Discovery
Each service gets a DNS name equal to its service name (e.g., myapp_node-app).
Containers in the same Swarm network can resolve services by name without knowing IPs.
Nginx and other services can route traffic using these internal service names.


## **Services**

| Service        | Port       | Description                                                |
|----------------|-----------|------------------------------------------------------------|
| Node app       | 3000      | NestJS app, responds at `/hello` and `/metrics`           |
| Python app     | 8000      | FastAPI app, responds at `/health` and `/metrics`         |
| Nginx          | 80        | Routes client-a → Node app, client-b → Python app         |
| cAdvisor       | 8081      | Container metrics (CPU, memory, network)                  |
| Prometheus     | 9090      | Collects metrics from Node, Python, and cAdvisor          |
| Grafana        | 3001      | Dashboard to visualize metrics                             |

---

## **Nginx Virtual Hosts**

| Host                 | Target Service          |
|----------------------|-----------------------|
| client-a.example.com  | Node app (`/hello`)   |
| client-b.example.com  | Python app (`/health`) |

> Use `/etc/hosts` mapping to test in browser:

<EC2-IP> client-a.example.com
<EC2-IP> client-b.example.com


---

## **Setup & Run**

### 1. Deploy stack in Swarm

```bash
docker stack deploy -c docker-compose.yml myapp
2. Verify services
docker service ls
docker service ps myapp_nginx
Testing Services
From Browser

Node app: http://client-a.example.com/ → {"message":"Hello from NestJS"}
Python app: http://client-b.example.com/ → {"status":"ok","service":"fastapi-app"}

Grafana: http://<EC2-IP>:3001 → login admin/admin
Prometheus: http://<EC2-IP>:9090 → check /targets
cAdvisor: http://<EC2-IP>:8081 → container metrics

From Host (curl)
curl -H "Host: client-a.example.com" http://localhost/
curl -H "Host: client-b.example.com" http://localhost/
curl http://localhost:3001
curl http://localhost:9090/targets
curl http://localhost:8081/metrics
From Node/Python container
docker exec -it <container_id> sh
curl http://myapp_node-app:3000/hello
curl http://myapp_python-app:8000/health
curl http://myapp_grafana:3000
curl http://<EC2-IP>:9090/targets  # if Prometheus not on same network
Prometheus Metrics Endpoints
App	Metrics URL
Node app	/metrics (NestJS with prom-client)
Python app	/metrics (FastAPI with prometheus_client)
cAdvisor	/metrics
Notes

Nginx must be reloaded after any changes to nginx.conf:
docker service update --force myapp_nginx
Prometheus may need prometheus.yml updated if service names or ports change.
Ensure all containers are on the same overlay network if you want internal DNS resolution by service name.


