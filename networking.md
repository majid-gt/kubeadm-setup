# DevOps Networking Fundamentals

## Overview

This document explains the core networking concepts required to understand infrastructure, containers, and Kubernetes environments. It covers how machines communicate, how services are exposed, and how networking layers interact in cloud-native systems.

The guide moves from basic networking concepts to Kubernetes networking and the real architecture used in production DevOps environments.

---

# 1. What Networking Means

Networking refers to how computers communicate with each other across systems.

Example:

```
Laptop
  │
  │ Internet
  ▼
Cloud VM
  │
  ▼
Web Server
```

When you open a website in a browser, your computer sends a network request to a remote server.

Example request:

```
https://google.com
```

Your system must first determine the IP address of the domain before sending the request.

---

# 2. IP Address

An IP address uniquely identifies a machine on a network.

Example:

```
192.168.1.10
```

An IP address can be compared to a physical house address.

| Analogy | Networking |
|-------|-------|
| House Address | IP Address |

Example:

```
35.239.206.23
```

If a user opens:

```
http://35.239.206.23
```

the request will reach the machine with that IP.

---

# 3. Types of IP Addresses

## Public IP

Accessible from the internet.

Example:

```
34.72.12.90
```

## Private IP

Used within internal networks.

Examples:

```
10.x.x.x
172.16.x.x – 172.31.x.x
192.168.x.x
```

Example inside Kubernetes:

```
10.96.0.10
```

---

# 4. Ports

A port represents a communication endpoint for a specific service running on a machine.

A single machine can run multiple services simultaneously using different ports.

Example ports:

| Port | Service |
|-----|------|
| 22 | SSH |
| 80 | HTTP |
| 443 | HTTPS |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 3000 | Grafana |
| 9090 | Prometheus |

Example connection:

```
34.72.12.90:80
```

This means a web server running on port 80 of that machine.

---

# 5. Protocols

Protocols define how data is transmitted over networks.

## TCP

Reliable communication protocol.

Used by:

```
HTTP
HTTPS
SSH
PostgreSQL
```

Example:

```
HTTP → TCP port 80
```

## UDP

Faster but less reliable protocol.

Used by:

```
DNS
Streaming
Gaming
```

Example:

```
DNS → UDP port 53
```

---

# 6. What Happens When You Open a Website

Example request:

```
https://google.com
```

Process:

```
Browser
  │
  ▼
DNS resolves domain
  │
  ▼
Returns IP address
  │
  ▼
Browser connects to IP:443
  │
  ▼
Web server responds
```

Example resolution:

```
google.com → 142.250.182.14
```

Connection occurs at:

```
142.250.182.14:443
```

---

# 7. Firewalls

A firewall controls which network traffic is allowed to reach a system.

Example firewall rules:

```
Allow TCP 22
Allow TCP 80
Allow TCP 443
Block everything else
```

If port 80 is blocked, the website cannot be accessed.

Example rule in Google Cloud:

```
Allow TCP 30000-32767
```

This range is used by Kubernetes NodePort services.

---

# 8. Servers

A server is simply a machine running services that respond to network requests.

Example:

```
VM
├── nginx (port 80)
├── postgres (port 5432)
└── ssh (port 22)
```

Clients send requests to these services.

---

# 9. Reverse Proxy

A reverse proxy receives incoming requests and forwards them to backend services.

Example:

```
Internet
   │
   ▼
NGINX
   │
   ├── /        → frontend
   ├── /api     → backend
   └── /admin   → application
```

In Kubernetes this role is performed by an Ingress Controller.

---

# 10. Networking Inside a Virtual Machine

Example installation of Nginx:

```
sudo apt install nginx
```

Nginx listens on:

```
port 80
```

Traffic flow:

```
Internet
  │
Firewall
  │
VM
  │
Port 80
  │
Nginx
```

---

# 11. Container Networking

Containers expose ports which can be mapped to the host system.

Example:

```
docker run -p 3000:3000 grafana
```

Meaning:

```
HOST PORT → CONTAINER PORT
3000 → 3000
```

Access Grafana using:

```
http://VM_IP:3000
```

---

# 12. Kubernetes Networking Basics

In Kubernetes every Pod receives its own IP address.

Example:

```
Pod A → 10.244.1.5
Pod B → 10.244.2.7
```

Pod IPs are temporary and change when pods restart.

To provide stable networking Kubernetes uses Services.

---

# 13. Kubernetes Service

A Service provides a stable network endpoint for pods.

Example:

```
Service: backend
IP: 10.96.10.20
```

Other pods connect using the service name:

```
backend:8000
```

instead of the pod IP.

---

# 14. Types of Kubernetes Services

## ClusterIP

Default service type.

Accessible only within the cluster.

Example architecture:

```
Frontend Pod
   │
   ▼
backend ClusterIP
   │
   ▼
Backend Pod
```

External users cannot access ClusterIP services.

---

## NodePort

Exposes a service on each node's IP.

Port range:

```
30000 – 32767
```

Example:

```
grafana → NodePort 30301
```

Access using:

```
http://NODE_IP:30301
```

Architecture:

```
Internet
  │
Node IP
  │
NodePort
  │
Service
  │
Pod
```

---

## LoadBalancer

Used in cloud environments.

Cloud provider creates an external load balancer.

Example providers:

```
AWS ELB
GCP Load Balancer
Azure Load Balancer
```

Architecture:

```
Internet
   │
Cloud Load Balancer
   │
Kubernetes Service
   │
Pods
```

These services usually incur additional cloud costs.

---

## Ingress

Ingress provides HTTP routing for multiple services using a single entry point.

Example routing:

```
example.com
grafana.example.com
prometheus.example.com
```

Architecture:

```
Internet
  │
Ingress Controller
  │
Routes traffic
  │
Services
  │
Pods
```

Common controllers:

```
NGINX
Traefik
HAProxy
```

---

# 15. Traffic Flow in the Application

Example architecture used in the project:

```
User Browser
   │
   ▼
Domain DNS
   │
   ▼
VM Public IP
   │
   ▼
Firewall
   │
   ▼
Ingress Controller
   │
   ├── / → frontend
   ├── /api → backend
   ├── /monitoring → grafana
   │
   ▼
Services
   │
   ▼
Pods
```

---

# 16. Pod-to-Pod Communication

Kubernetes allows all pods to communicate with each other by default.

Example:

```
Pod A → Pod B
```

Communication can occur using:

- Pod IP
- Service DNS name

Example:

```
http://backend-service:8000
```

---

# 17. Ports vs TargetPorts

Kubernetes services use multiple port definitions.

Example:

```
port: 80
targetPort: 8000
nodePort: 30080
```

Meaning:

```
Client connects → port 80
Kubernetes forwards → container port 8000
```

---

# 18. Kubernetes Internal Networking

Example communication inside cluster:

```
frontend pod
   │
   ▼
backend service
   │
   ▼
backend pod
```

Pods communicate using service DNS names.

---

# 19. Networking Layers

Networking operates in layered models.

| Layer | Example |
|------|------|
| Application | HTTP |
| Transport | TCP |
| Network | IP |
| Link | Ethernet |

Example request stack:

```
HTTP → TCP → IP → Ethernet
```

---

# 20. Common DevOps Networking Tools

Useful commands for debugging networking.

Check services:

```
kubectl get svc
```

Check ingress:

```
kubectl get ingress
```

Check pods with IP addresses:

```
kubectl get pods -o wide
```

Check open ports:

```
netstat -tulnp
```

Connectivity tests:

```
curl
ping
telnet
```

---

# 21. Networking Concepts Used in the Project

The project already uses many real-world networking components:

```
DNS
Firewall rules
Ports
Ingress
ClusterIP
NodePort
Reverse proxy
TLS
Load balancing
```

---

# 22. DevOps Networking Mental Model

Understanding networking becomes easier if viewed as a layered flow:

```
Internet
   │
   ▼
DNS
   │
   ▼
Firewall
   │
   ▼
Load Balancer
   │
   ▼
Ingress
   │
   ▼
Service
   │
   ▼
Pods
```

---

# 23. Typical Production Networking Architecture

Most production systems follow a similar architecture:

```
Internet
   │
Cloud Load Balancer
   │
Ingress Controller
   │
Services
   │
Pods
   │
Databases
```

---

# 24. Topics to Learn Next

Important networking concepts for DevOps engineers include:

```
CIDR ranges
VPC networking
Subnets
NAT gateways
Service meshes
Network policies
```

---

# 25. Key Takeaway

Networking can feel complex because multiple layers interact simultaneously.

However the core flow remains consistent:

```
Domain → IP → Port → Firewall → Ingress → Service → Pod
```

Understanding this chain simplifies troubleshooting and system design in cloud-native environments.
