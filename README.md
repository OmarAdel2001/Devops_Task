# Scalable Backend with Kubernetes

This project demonstrates a scalable backend service deployed using Kubernetes. It includes a Node.js/Express CRUD API connected to a MongoDB Replica Set (3 nodes) deployed as a StatefulSet. The backend is configured with Horizontal Pod Autoscaling (HPA) and exposed via an Ingress.

## Architecture

1. **Backend API**: A Node.js and Express application exposing a simple RESTful CRUD API for 'Posts'.
2. **Database**: MongoDB deployed as a StatefulSet with 3 replicas configured as a Replica Set to ensure High Availability and no data loss.
3. **Containerization**: Docker is used to containerize the backend application.
4. **Kubernetes Resources**:
   - **Deployment**: Manages the backend pods (min: 1, max: 5).
   - **StatefulSet**: Manages the MongoDB nodes with persistent storage.
   - **HorizontalPodAutoscaler (HPA)**: Scales the backend deployment based on CPU utilization (target 70%).
   - **Services**: ClusterIP for internal communication. Headless service for MongoDB StatefulSet.
   - **Ingress**: Exposes the backend API externally.

## Prerequisites

- A Kubernetes Cluster (EKS on AWS, GKE on GCP, or Minikube for local testing).
- `kubectl` configured to communicate with your cluster.
- `docker` installed locally to build and push the image.
- An Ingress Controller installed in your cluster (e.g., NGINX Ingress Controller).
- Metrics Server installed (required for HPA to work).

## Setup Steps

### 1. Build and Push Docker Image

1. Build the Docker image:
   ```bash
   cd backend
   docker build -t omaradel2001/backend:latest .
   ```
2. Push the image to Docker Hub:
   ```bash
   docker push omaradel2001/backend:latest
   ```

### 2. Deploy MongoDB Replica Set

1. Apply the MongoDB manifests:
   ```bash
   kubectl apply -f k8s/mongodb/mongo-service.yaml
   kubectl apply -f k8s/mongodb/mongo-statefulset.yaml
   ```
2. Wait for the MongoDB pods to be ready (`kubectl get pods`).
3. Initialize the replica set by applying the init job:
   ```bash
   kubectl apply -f k8s/mongodb/mongo-init-job.yaml
   ```
   *(Note: The job will execute `rs.initiate()` on the first mongo node)*

### 3. Deploy Backend API

1. Apply the backend manifests:
   ```bash
   kubectl apply -f k8s/backend/
   ```
2. Ensure the pods are running and the HPA is active:
   ```bash
   kubectl get pods
   kubectl get hpa
   ```

### 4. Configure Ingress Host

To test locally via Ingress, add the host to your `/etc/hosts` file (or equivalent on Windows):
```
<INGRESS_CONTROLLER_EXTERNAL_IP> backend.local
```

---

## API Endpoints

The API is accessible at `http://backend.local` (or your ingress IP).

| Method | Endpoint      | Description |
|--------|--------------|-------------|
| GET    | `/health`    | Health check |
| GET    | `/posts`     | Get all posts |
| POST   | `/posts`     | Create a new post (Body: `{ "title": "...", "content": "..." }`) |
| GET    | `/posts/:id` | Get a post by ID |
| PUT    | `/posts/:id` | Update a post by ID |
| DELETE | `/posts/:id` | Delete a post by ID |
| GET    | `/stress`    | Simulates high CPU load to trigger HPA scaling |

---

## Video Recording Guide

For the final deliverable, record a video (Arabic/English) covering the following steps:

### 1. Architecture Explanation
- Briefly explain the components: Node.js API, MongoDB StatefulSet, Services, Ingress, and HPA.
- Show the Kubernetes manifests on your screen.

### 2. Running System Demo
- Show the pods running: `kubectl get pods`.
- Show the API working by sending a `POST` request to create a post, and a `GET` request to retrieve it (use Postman or `curl`).
- Verify the data is saved by connecting to the MongoDB database or simply retrieving it via the API.

### 3. Auto-scaling in Action
- Show the current HPA status: `kubectl get hpa backend-hpa --watch` (in a separate terminal).
- Trigger CPU load by calling the `/stress` endpoint multiple times or using a load testing tool like `hey` or Apache Benchmark (`ab -n 1000 -c 50 http://backend.local/stress`).
- Show the `TARGETS` CPU % increasing in the HPA terminal.
- Show the `REPLICAS` count increasing from 1 up to 5.
- Show `kubectl get pods` to see the new pods initializing.

### 4. High Availability (Failover) Test
- **Backend Pod Deletion**: 
  - Run `kubectl delete pod -l app=backend`.
  - Immediately send a request to the API. It should still work because Kubernetes quickly spins up a new pod (or if scaled, other pods handle the load).
- **MongoDB Pod Deletion**: 
  - Show the current posts.
  - Delete a MongoDB pod: `kubectl delete pod mongodb-0`.
  - Wait for it to restart, or immediately fetch posts again.
  - Explain that because it's a Replica Set (3 nodes) with persistent volumes, data is not lost, and the other nodes seamlessly take over primary election if needed. Show that the posts are still there.
