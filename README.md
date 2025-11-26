
#  Video-to-Audio Microservices Application

Convert videos to audio using a scalable microservices architecture deployed on **AWS EKS (Kubernetes)**.

---

## 🏗️ **Architecture Overview**

This project contains **4 microservices**:

| Service                  | Purpose                                                         | 
| ------------------------ | --------------------------------------------------------------- |
| **Auth Service**         | Handles login, JWT token generation, and user authentication    |
| **Gateway Service**      | Acts as API gateway and exposes login/upload/download endpoints |
| **Converter Service**    | Converts uploaded videos into MP3 format                        |
| **Notification Service** | Sends email notifications containing MP3 file ID                |

### **Databases & Message Queue**

| Component      | Purpose                                                         |
| -------------- | --------------------------------------------------------------- |
| **PostgreSQL** | Stores users, JWT auth details, and video metadata              |
| **MongoDB**    | Stores generated MP3 files                                      |
| **RabbitMQ**   | Async message queuing between converter & notification services |

---

## ☸️ **Kubernetes Cluster Setup (AWS EKS)**

### **1. Create IAM Roles**

You created:

* **EKS Cluster Role**
* **EKS Node Group Role**

### **2. Create EKS Cluster**

Cluster name: **Microservices**

### **3. Create Node Group**

Required for running pods.

### **4. Connect AWS CLI to Local**

```bash
aws configure
```

Added:

* Access Key
* Secret Key
* Region
* Output format

Since you used the **root account**, no extra EKS permissions were needed.

### **5. Update kubeconfig**

To connect kubectl → cluster:

```bash
aws eks update-kubeconfig --name Microservices --region us-east-1
```

---

## 📦 Install Required Services using Helm

### **MongoDB**

```bash
cd Helm_charts/MongoDB
helm install mongo .
```

MongoDB Pod & Service are created automatically.

#### Connect to MongoDB:

```bash
mongosh mongodb://<username>:<pwd>@<nodeIP>:30005/mp3s?authSource=admin
```

---

### **PostgreSQL**

```bash
cd ../Postgres
helm install postgres .
```

Connect:

```bash
psql "postgres://<username>:<pwd>@<nodeIP>:30003/authdb"
```

---

### **RabbitMQ**

```bash
helm install rabbitmq .
```

Access UI:

```
http://<nodeIP>:<rabbitmq_port>
```

---

## 🔐 Important Security Group Updates

On the Node Group EC2 instance, you added inbound rules for:

| Port      | Purpose         |
| --------- | --------------- |
| **30005** | MongoDB         |
| **30004** | RabbitMQ        |
| **30003** | PostgreSQL      |
| **30002** | Gateway Service |

---

## 🐳 Building & Pushing Docker Images

### Example for Auth Service:

```bash
docker build -t auth .
docker tag auth:latest ayushipanwar24/auth
docker push ayushipanwar24/auth
```

Repeat for:

* auth-service
* gateway-service
* converter-service
* notification-service

---

## 🚀 Deploy Microservices to Kubernetes

Go to each manifest folder:

### Auth Service

```bash
cd auth-service/manifest
kubectl apply -f .
```

### Gateway Service

```bash
cd gateway-service/manifest
kubectl apply -f .
```

### Converter Service

```bash
cd converter-service/manifest
kubectl apply -f .
```

### Notification Service

```bash
cd notification-service/manifest
kubectl apply -f .
```

### Verify Deployment

```bash
kubectl get all
```

Each service runs in its own **separate pod**, which allows:

* Independent scaling
* Independent failure recovery
* Loose coupling between services

(You SHOULD NOT run multiple microservices inside one pod.)

---

## 🔗 **API Endpoints (Via Gateway)**

Gateway Service is exposed on:

```
http://<nodeIP>:30002/
```

---

### ### 🔐 **1. Login Endpoint**

**POST**

```
http://nodeIP:30002/login
```

**cURL:**

```bash
curl -X POST http://nodeIP:30002/login -u <email>:<password>
```

Expected output:

```
success!
```

---

### 📤 **2. Upload Video Endpoint**

Uploads video → stored in PostgreSQL → sent to converter → converted → stored in MongoDB.

**POST**

```
http://nodeIP:30002/upload
```

**cURL:**

```bash
curl -X POST -F "file=@./video.mp4" \
-H "Authorization: Bearer <JWT_TOKEN>" \
http://nodeIP:30002/upload
```

You will receive:

* An **email** containing the MP3 File ID

---

### 📥 **3. Download MP3 Endpoint**

Retrieve converted MP3 from MongoDB.

**GET**

```
http://nodeIP:30002/download?fid=<Generated ID>
```

**cURL:**

```bash
curl --output video.mp3 -X GET \
-H "Authorization: Bearer <JWT_TOKEN>" \
"http://nodeIP:30002/download?fid=<Generated ID>"
```

---

## 🚀 Workflow Summary

1. User logs in → receives JWT.
2. User uploads video.
3. Video metadata stored in PostgreSQL.
4. Gateway pushes message to RabbitMQ.
5. Converter service processes the video → generates MP3 → stores in MongoDB.
6. Notification service sends email with MP3 ID.
7. User downloads MP3 via download endpoint.

---

## 🎯 Key Benefits of Microservices in This Project

* Independent scaling of each service
* Fault isolation
* Technology flexibility (Node.js + Python)
* Easy CI/CD integration
* High availability via Kubernetes


