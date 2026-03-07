
# 🚀 3-Tier CRUD Application

This repository demonstrates the deployment of a robust **3-Tier CRUD (Create, Read, Update, Delete)** application. It showcases various infrastructure strategies, ranging from manual setup and cloud integration (S3) to containerization using Docker and orchestration using Kubernetes.

## 📚 Deployment Guides

This repository includes detailed guides for different deployment scenarios. Please choose the one that fits your requirements:

| # | Deployment Strategy | Description | Guide |
| :--- | :--- | :--- | :--- |
| 1 | **Manual Setup** | Step-by-step setup of servers and components manually. Ideal for learning the basics of the architecture. | [**3Tier-Architecture(manual).md**](./3Tier-Architecture(manual).md) |
| 2 | **Manual + AWS S3** | Manual setup integrated with **AWS S3** for object storage. Best for cloud-based static asset handling. | [**3-Tier-architecture (With S3).md**](./3-Tier-architecture%20(With%20S3).md) |
| 3 | **Docker (Docker Compose)** | Containerized deployment using a YAML file. Perfect for quick local testing or single-host deployments. | [**3-Tier_EasyCRUD_(using_Docker.yaml).md**](./3-Tier_EasyCRUD_(using_Docker.yaml).md) |
| 4 | **Kubernetes (K8s)** | Production-grade deployment using Kubernetes. Best for scalability, auto-healing, and managing containerized workloads. | [**3-Tier_EasyCRUD_(using_K8S).md**](./3-Tier_EasyCRUD_(using_K8S).md) |

---

## 🏗️ Architecture Overview

This project follows a standard **3-Tier Architecture** pattern:

1.  **Presentation Tier (Frontend):** The user interface.
2.  **Application Tier (Backend):** The business logic and API server.
3.  **Data Tier (Database):** The database management system (and S3 for file storage in the specific guide).

## ⚙️ Prerequisites

*   **Git:** To clone this repository.
*   **Basic CLI Knowledge:** Familiarity with the terminal.
*   **Tools:** Depends on the guide you follow (e.g., Docker & Docker Compose, kubectl, or AWS CLI).


