# Spring Boot Automated CI/CD Pipeline

A containerized Java Spring Boot application featuring a fully automated GitOps continuous integration and continuous deployment (CI/CD) lifecycle. Code changes pushed to GitHub are automatically built, containerized, and deployed to a multi-node Kubernetes cluster managed via Rancher.

---

## 🏗️ System Architecture

```text
[Local Windows PC] --- (Git Push) ---> [GitHub Repository]
                                               │
                                        (Webhook/Poll)
                                               ▼
[Rancher App URL] <--- (Deploy) --- [Jenkins Automation]
       │                                       │
 (NodePort 32085)                       (Maven Build)
       ▲                                       │
       │                                (Docker Build)
 [K8s Cluster] <--- (Pull Image) ◄─────────────┘
