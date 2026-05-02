# Project Architecture & Component Explanation

This document provides a detailed explanation of every component built for the Scalable Backend project. You can use this as a script or study guide for your **Architecture Explanation** video deliverable.

---

## 1. The Backend API (`backend/index.js` & `backend/models/Post.js`)
**What it is:** A Node.js application using the Express framework.
**Why we used it:** It is lightweight, fast, and perfect for building simple REST APIs. 
- **Mongoose / Post.js:** We use Mongoose to define a schema (`Post.js`) which ensures all posts have a `title` and `content`.
- **CRUD Operations:** The `index.js` file exposes endpoints to Create (`POST`), Read (`GET`), Update (`PUT`), and Delete (`DELETE`) posts.
- **Stress Endpoint (`/stress`):** We built a custom endpoint that runs a massive mathematical loop to intentionally spike the CPU. This is how we prove that the Auto-scaling (HPA) works.
- **Database Connection:** Instead of connecting to a single database IP, it connects to a **MongoDB Replica Set URI**, which contains the addresses of all 3 database nodes. If one node dies, Mongoose automatically talks to the next available one.

---

## 2. Dockerization (`backend/Dockerfile`)
**What it is:** A blueprint to package our Node.js application.
**Why we used it:** It ensures our application runs exactly the same way on any machine or cloud provider.
- `FROM node:18-alpine`: We use the "alpine" version of Node because it is a highly compressed, tiny operating system, making our Docker image very small and fast to download.
- `COPY package*.json ./` & `RUN npm install`: We install the dependencies.
- `CMD ["npm", "start"]`: The command that starts the server when the container runs in Kubernetes.

---

## 3. Database: MongoDB StatefulSet (`k8s/mongodb/`)
In Kubernetes, standard "Deployments" are for stateless apps (like our backend). Databases require permanent storage and strict identity, which is why we use a **StatefulSet**.

- **StatefulSet (`mongo-statefulset.yaml`)**:
  - `replicas: 3`: It creates exactly 3 pods (`mongodb-0`, `mongodb-1`, `mongodb-2`).
  - `volumeClaimTemplates`: It automatically creates a separate Persistent Volume (hard drive) for each pod so that if a pod is deleted, its data remains safe on the hard drive.
  - `--replSet rs0`: A command passed to MongoDB telling it to run in Replica Set mode rather than standalone.
- **Headless Service (`mongo-service.yaml`)**: Unlike normal services that balance traffic randomly, a headless service (`clusterIP: None`) gives a unique DNS name to *each individual pod* (e.g., `mongodb-0.mongodb-headless...`). This is mandatory for a MongoDB Replica Set so the nodes can talk to each other directly to elect a primary database.
- **Init Job (`mongo-init-job.yaml`)**: Because the 3 MongoDB nodes start empty, they don't know they belong to a cluster. This temporary job runs a command (`rs.initiate()`) against `mongodb-0` to link all three nodes together into a highly available Replica Set.

---

## 4. Backend Deployment (`k8s/backend/backend-deployment.yaml`)
**What it is:** The Kubernetes controller that manages our Node.js pods.
- `replicas: 1`: We start with a minimum of 1 pod.
- `image: omaradel2001/backend:latest`: It pulls your specific Docker image.
- `resources (requests/limits)`: This is crucial! We explicitly tell Kubernetes that this pod normally uses `100m` of CPU (10% of a CPU core), but can't exceed `500m`. Without these limits, the Auto-scaler (HPA) cannot calculate percentages and wouldn't work.

---

## 5. Horizontal Pod Autoscaler (`k8s/backend/backend-hpa.yaml`)
**What it is:** The HPA automatically increases or decreases the number of backend pods based on traffic.
- `minReplicas: 1` and `maxReplicas: 5`: The bounds for scaling.
- `averageUtilization: 70`: It continuously monitors the pods. If the average CPU usage across all pods goes above 70%, it tells the Deployment to create more pods to handle the load. When traffic drops, it slowly terminates the extra pods to save money.

---

## 6. Services & Ingress (`k8s/backend/`)
- **Backend Service (`backend-service.yaml`)**: It is a `ClusterIP` service. It groups all of our backend pods together and provides a single internal IP address and port (`80`) to reach them. It also acts as a load balancer, distributing requests evenly among all available backend pods.
- **Ingress (`backend-ingress.yaml`)**: A ClusterIP service is only reachable *inside* the cluster. To let outside users access the API, we use an Ingress. It acts as an API Gateway/Router, taking traffic from `backend.local` on the public internet and routing it strictly to the internal `backend-service`.

---

## Summary of High Availability & Failover
Because of this architecture, we have achieved High Availability:
1. **If a Backend Pod dies:** The Kubernetes `Deployment` instantly spins up a new one to maintain the desired replica count. Because the `Service` load balances, users barely notice a blip.
2. **If a MongoDB Pod dies:** The data is safely stored on the Persistent Volume. The Replica Set immediately realizes the Primary node is gone, holds an election in milliseconds, and promotes a Secondary node to Primary. Our Node.js app automatically detects the new Primary and continues serving data without data loss.
