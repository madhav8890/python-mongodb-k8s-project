# Flask MongoDB Kubernetes Deployment

## Overview

This project demonstrates the deployment of a Python Flask application that connects to a MongoDB database using Kubernetes. The deployment includes autoscaling, persistent storage, DNS resolution, and resource management.

## Prerequisites

- Python 3.8 or later
- Docker
- Kubernetes CLI (kubectl)
- Minikube or any other local Kubernetes cluster solution
- Docker Hub account (for pushing Docker images)

## Part 1: Docker Setup

### Dockerfile for Flask Application

Create a `Dockerfile` in the root of your project with the following content:

```Dockerfile
# Use the official Python image from the Docker Hub
FROM python:3.8-slim

# Set the working directory in the container
WORKDIR /app

# Copy the requirements file into the container at /app
COPY requirements.txt .

# Install the Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy the rest of the application code into the container
COPY . .

# Set the environment variable for Flask
ENV FLASK_APP=app.py

# Expose the port that the Flask app will run on
EXPOSE 5000

# Command to run the Flask app
CMD ["flask", "run", "--host=0.0.0.0", "--port=5000"]
```

### Building and Pushing the Docker Image

1. **Build the Docker Image:**

   ```bash
   docker build -t madhav01/flask-mongodb-app:latest .
   ```

2. **Login to Docker Hub:**

   ```bash
   docker login
   ```

3. **Push the Docker Image:**

   ```bash
   docker push madhav01/flask-mongodb-app:latest
   ```

## Part 2: Kubernetes Setup

### Kubernetes YAML Files

#### 1. Deployment for Flask Application (`flask-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
      - name: flask-container
        image: madhav01/flask-mongodb-app:latest
        ports:
        - containerPort: 5000
        resources:
          requests:
            memory: "250Mi"
            cpu: "200m"
          limits:
            memory: "500Mi"
            cpu: "500m"
        env:
        - name: MONGODB_URI
          value: "mongodb://mongo-user:mongo-password@mongodb-service:27017/"
```

#### 2. StatefulSet for MongoDB (`mongo-statefulset.yaml`)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: "mongodb-service"
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
      - name: mongo
        image: mongo:latest
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: mongo-persistent-storage
          mountPath: /data/db
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          value: "mongo-user"
        - name: MONGO_INITDB_ROOT_PASSWORD
          value: "mongo-password"
  volumeClaimTemplates:
  - metadata:
      name: mongo-persistent-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

#### 3. Services for Flask and MongoDB (`services.yaml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-service
spec:
  selector:
    app: flask-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
  type: NodePort

---

apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  selector:
    app: mongo
  ports:
    - protocol: TCP
      port: 27017
  clusterIP: None
```

#### 4. Horizontal Pod Autoscaler (HPA) (`hpa.yaml`)

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: flask-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: flask-deployment
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70
```

### Deployment Steps on Minikube

1. **Start Minikube:**

   ```bash
   minikube start
   ```

2. **Apply the MongoDB StatefulSet and Service:**

   ```bash
   kubectl apply -f mongo-statefulset.yaml
   kubectl apply -f services.yaml
   ```

3. **Apply the Flask Deployment and Service:**

   ```bash
   kubectl apply -f flask-deployment.yaml
   ```

4. **Apply the Horizontal Pod Autoscaler:**

   ```bash
   kubectl apply -f hpa.yaml
   ```

5. **Access the Flask Application:**

   - Get the Minikube IP:
   
     ```bash
     minikube ip
     ```
   
   - Get the NodePort of the Flask Service:
   
     ```bash
     kubectl get services flask-service
     ```
   
   - Access the Flask application at `http://<Minikube_IP>:<NodePort>`.

### DNS Resolution in Kubernetes

In Kubernetes, each service is assigned a DNS name, making it accessible within the cluster using that name. The DNS resolution is handled by Kubernetes’ internal DNS service, which maps the service name to the appropriate IP address.

For instance, the Flask application connects to MongoDB using the DNS name `mongodb-service`. Kubernetes resolves this name to the internal IP of the MongoDB pod, enabling communication between the services.

### Resource Requests and Limits in Kubernetes

- **Resource Requests:** Specify the minimum resources required by a container to run efficiently. Kubernetes ensures that a node has at least these resources available before scheduling the pod.
  
- **Resource Limits:** Specify the maximum resources a container can use. This prevents a container from consuming too many resources and impacting other pods.

**Example:**

- Requests: `memory: 250Mi, cpu: 200m`
- Limits: `memory: 500Mi, cpu: 500m`

These settings ensure efficient resource utilization and cluster stability.

### Design Choices

- **StatefulSet for MongoDB:** Chosen because MongoDB requires persistent storage and consistent network identifiers. StatefulSets are ideal for databases that require these characteristics.
  
- **NodePort for Flask Service:** Chosen for ease of access during local development and testing with Minikube. Alternatively, a LoadBalancer service could be used in a cloud environment.

- **Horizontal Pod Autoscaler (HPA):** Ensures that the Flask application scales automatically based on CPU utilization, providing resilience under varying loads.

### Testing Scenarios

#### 2. Database Interaction

- Test inserting and retrieving data using the `/data` endpoint to verify that the Flask application interacts correctly with MongoDB.

#### 3. Persistence

- Delete the MongoDB pod and verify that the data persists by checking the data after the pod is recreated. This ensures that the persistent volume is functioning as expected.

### Conclusion

This project provides a comprehensive deployment of a Python Flask application with MongoDB on Kubernetes, covering essential features like autoscaling, persistent storage, DNS resolution, and resource management. The setup is designed to be robust and scalable, ensuring that the application can handle varying loads efficiently.

