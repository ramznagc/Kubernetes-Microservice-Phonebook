# Hands-on Kubernetes-206: Microservice Phonebook Application on Kubernetes

## Purpose

The purpose of this hands-on project is to deploy a microservice-based Phonebook web application using Python Flask and MySQL on a Kubernetes cluster. You will learn how to containerize applications, create Kubernetes manifests, manage secrets and configmaps, and use persistent storage.

---

## Learning Outcomes

At the end of this hands-on, you will be able to:

- Build and push Docker images for microservices.
- Write Kubernetes manifests for Deployments, Services, Persistent Volumes, Secrets, and ConfigMaps.
- Deploy and manage a multi-container application on Kubernetes.
- Use persistent storage for databases in Kubernetes.
- Secure sensitive data using Kubernetes Secrets.
- Expose services using NodePort and ClusterIP.
- Access the application from a browser.

---

## Outline

- Part 1: Prepare the Environment
- Part 2: Build and Push Docker Images
- Part 3: Create Kubernetes Manifests
- Part 4: Deploy MySQL Database with Persistent Storage
- Part 5: Deploy Web and Result Servers
- Part 6: Expose Services and Test the Application

---

## Part 1: Prepare the Environment

### 1.1. Launch Your Kubernetes Cluster with Terraform

- Use a provided or custom Terraform configuration to provision your Kubernetes cluster infrastructure (e.g., EC2 instances, networking, security groups) on AWS.
- Ensure your cluster has at least two nodes (one master, one worker) and meets the minimum requirements (e.g., t2.medium or larger).
- Once Terraform completes, your Kubernetes cluster should be ready for application deployment.

---

## Part 2: Build and Push Docker Images

### 2.1. Prepare Application Code

- Download or clone the project code from the GitHub repository.

### 2.2. Build Docker Images

- Navigate to the `image_for_web_server` directory and build the Docker image:

```bash
cd Students_files/image_for_web_server
docker build -t <your-dockerhub-username>/web-server:latest .
```

- Navigate to the `image_for_result_server` directory and build the Docker image:

```bash
cd ../image_for_result_server
docker build -t <your-dockerhub-username>/result-server:latest .
```

### 2.3. Push Images to Docker Hub

```bash
docker login
docker push <your-dockerhub-username>/web-server:latest
docker push <your-dockerhub-username>/result-server:latest
```

---

## Part 3: Create Kubernetes Manifests

### 3.1. Create Secrets and ConfigMaps

Below are example YAMLs for the required Kubernetes Secrets and ConfigMaps. **Remember to replace base64-encoded values with your own secure credentials.**

**app-secret.yaml**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  labels:
    app: phonebook-app
  # This secret is for the Flask app's environment variables
  # Use: echo -n 'yourvalue' | base64 to encode values

type: Opaque
data:
  MYSQL_PASSWORD: Q2xhcnVzd2F5XzE= 

```

**mysql-secret.yaml**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  labels:
    app: mysql

type: Opaque
data:
  MYSQL_ROOT_PASSWORD: UjEyMzQ1cg==
  MYSQL_PASSWORD: Q2xhcnVzd2F5XzE=
```

**database_configmap.yaml**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: database-config
data:
  MYSQL_DATABASE_HOST: mysql
  MYSQL_DATABASE: phonebook
  MYSQL_USER: clarus

```

---

### 3.2. Create Persistent Volume and Claim

**persistent_volume.yaml**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data
```

**persistent_volume_claim.yaml**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

---

### 3.3. Create Deployment and Service Manifests

**mysql_deployement.yml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  replicas: 1
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        envFrom:
        - configMapRef:
            name: database-config
        - secretRef:
            name: mysql-secret
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
```

**mysql_service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  selector:
    app: mysql
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
  type: ClusterIP
```

**web_server_deployment.yml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-server
  template:
    metadata:
      labels:
        app: web-server
    spec:
      containers:
      - name: web-server
        image: 01447/web-server:latest
        envFrom:
        - configMapRef:
            name: database-config
        - secretRef:
            name: app-secret
        ports:
        - containerPort: 80
```

**web_server_service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-server
spec:
  selector:
    app: web-server
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30001
  type: NodePort
```

**result_server_deployment.yml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: result-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: result-server
  template:
    metadata:
      labels:
        app: result-server
    spec:
      containers:
      - name: result-server
        image: 01447/result-server:latest
        envFrom:
        - configMapRef:
            name: database-config
        - secretRef:
            name: app-secret
        ports:
        - containerPort: 80
```

**result_server_service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: result-server
spec:
  selector:
    app: result-server
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30002
  type: NodePort
```

---

**Note:**
- Replace `<your-dockerhub-username>` with your actual Docker Hub username.
- The base64 values in the secret are for demonstration; use your own secure values and encode them with `echo -n 'yourvalue' | base64`.

---

## Part 4: Deploy MySQL Database with Persistent Storage

### 4.1. Apply Persistent Volume and Claim

```bash
kubectl apply -f persistent_volume.yaml
kubectl apply -f persistent_volume_claim.yaml
```

### 4.2. Apply Secrets and ConfigMaps

```bash
kubectl apply -f mysql-secret.yaml
kubectl apply -f database_configmap.yaml
kubectl apply -f servers_configmap.yaml
```

### 4.3. Deploy MySQL

```bash
kubectl apply -f mysql_deployement.yml
kubectl apply -f mysql_service.yaml
```

- Check the status:

```bash
kubectl get pods
kubectl get svc
```

---

## Part 5: Deploy Web and Result Servers

### 5.1. Deploy Web Server

```bash
kubectl apply -f web_server_deployment.yml
kubectl apply -f web_server_service.yaml
```

### 5.2. Deploy Result Server

```bash
kubectl apply -f result_server_deployment.yml
kubectl apply -f result_server_service.yaml
```

- Check the status:

```bash
kubectl get pods
kubectl get svc
```

---

## Part 6: Expose Services and Test the Application

### 6.1. Access the Application

- Find the public IP of your EC2 instance (master or worker).
- Access the web server (add/update/delete) at:

```
http://<ec2-public-ip>:30001
```

- Access the result/search server at:

```
http://<ec2-public-ip>:30002
```

### 6.2. Test Functionality

- Add, update, and delete records using the web server.
- Search for records using the result server.
- Verify data persists after pod restarts (due to persistent volume).

---

## Clean Up

- To delete all resources:

```bash
kubectl delete -f .
```

---

## Expected Outcome

- The Phonebook application is running on Kubernetes.
- You can add, update, delete, and search records via the web interface.
- Data is persisted in MySQL using a persistent volume.
- Application is accessible from anywhere via NodePort services.

---

## Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [Docker Documentation](https://docs.docker.com/) 