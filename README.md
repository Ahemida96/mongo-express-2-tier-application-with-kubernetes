# Kubernetes Deployment for a 2-Tier Application (ExpressJS and MongoDB)

## Description
  Deploying a 2-tier application on a Kubernetes cluster. The application consists of an ExpressJS frontend and a MongoDB backend. The MongoDB instance is secured with authentication, and the ExpressJS application is configured to connect to the MongoDB database. Both components are deployed as separate pods within the Kubernetes cluster. This application also includes enhancements such as persistent storage, security with Kubernetes Secrets, health checks, Horizontal Pod Autoscaler (HPA), and Ingress

```plaintext
        +---------------------------------------------------------------+
        |                         Kubernetes Cluster                    |
        |                                                               |
        |  +------------------------+        +---------------------+    |
        |  |  Namespace: ahmedhemida|        |  Namespace: default |    |
        |  +------------------------+        +---------------------+    |
        |                                                               |
        |  +---------------------------------------------------------+  |
        |  |                         Ingress                         |  |
        |  |  - Host: mekn-app.ahmedhemida.com                       |  |
        |  |  - Routes traffic to mongo-express Service              |  |
        |  +---------------------------------------------------------+  |
        |                                                               |
        |  +----------------------+        +-------------------------+  |
        |  |  ExpressJS App       |        |  MongoDB Database       |  |
        |  |  - Deployment        |        |  - Deployment           |  |
        |  |  - Service: NodePort |        |  - Service: ClusterIP   |  |
        |  |  - Liveness/Readiness|        |  - PersistentVolume     |  |
        |  |  - HPA (Auto-scaling)|        |  - PersistentVolumeClaim|  |
        |  +----------------------+        +-------------------------+  |
        |                                                               |
        |  +---------------------------------------------------------+  |
        |  |                         Secrets                         |  |
        |  |  - MongoDB Credentials (user/pass)                      |  |
        |  |  - ExpressJS Basic Auth (user/pass)                     |  |
        |  +---------------------------------------------------------+  |
        |                                                               |
        +---------------------------------------------------------------+
```
## Steps

### 1. Create a Dedicated Namespace
Create a dedicated namespace for the application to isolate resources.

**File: `namespace.yaml`**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ahmedhemida
```

Apply the namespace:
```sh
kubectl apply -f namespace.yaml
```

---

### 2. Deploy MongoDB with Persistent Storage
Deploy MongoDB with persistent storage to ensure data persistence across pod restarts.

**File: `mongo-storage.yaml`**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
  namespace: ahmedhemida
spec:
  storageClassName: "rook-cephfs"
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

**File: `database-deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mekn-database
  namespace: ahmedhemida
  labels:
    app: mekn
    tier: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mekn
      tier: database
  template:
    metadata:
      labels:
        app: mekn
        tier: database
    spec:
      containers:
        - name: mongodb
          image: mongodb/mongodb-community-server
          ports:
            - containerPort: 27017
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mekn-secrets
                  key: mongo-user
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mekn-secrets
                  key: mongo-pass
          volumeMounts:
            - name: mongo-storage
              mountPath: /data/db
      volumes:
        - name: mongo-storage
          persistentVolumeClaim:
            claimName: mongo-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
  namespace: ahmedhemida
  labels:
    app: mekn
    tier: database
spec:
  type: ClusterIP
  ports:
    - port: 27017
      targetPort: 27017
  selector:
    app: mekn
    tier: database
```

Apply the MongoDB resources:
```sh
kubectl apply -f mongo-storage.yaml
kubectl apply -f database-deployment.yaml
```

---

### 3. Deploy ExpressJS Application with Health Checks and HPA
Deploy the ExpressJS application with health checks, Kubernetes Secrets, and Horizontal Pod Autoscaler (HPA).

**File: `secrets.yaml`**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mekn-secrets
  namespace: ahmedhemida
type: Opaque
data:
  mongo-user: dXNlcg==  
  mongo-pass: cGFzcw==  
  express-user: dXNlcg==  # base64 encoded "user"
  express-pass: cGFzcw==  # base64 encoded "pass"
```

**File: `app-deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mekn-backend
  namespace: ahmedhemida
  labels:
    app: mekn
    tier: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mekn
      tier: backend
  template:
    metadata:
      labels:
        app: mekn
        tier: backend
    spec:
      containers:
        - name: express-app
          image: mongo-express
          ports:
            - containerPort: 8081
          env:
            - name: ME_CONFIG_MONGODB_ADMINUSERNAME
              valueFrom:
                secretKeyRef:
                  name: mekn-secrets
                  key: mongo-user
            - name: ME_CONFIG_MONGODB_ADMINPASSWORD
              valueFrom:
                secretKeyRef:
                  name: mekn-secrets
                  key: mongo-pass
            - name: ME_CONFIG_MONGODB_SERVER
              value: mongodb-service
            - name: ME_CONFIG_BASICAUTH_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mekn-secrets
                  key: express-user
            - name: ME_CONFIG_BASICAUTH_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mekn-secrets
                  key: express-pass
          livenessProbe:
            httpGet:
              path: /
              port: 8081
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: 8081
            initialDelaySeconds: 5
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-express
  namespace: ahmedhemida
  labels:
    app: mekn
    tier: backend
spec:
  type: NodePort
  ports:
    - port: 8081
      targetPort: 8081
      nodePort: 32000
  selector:
    app: mekn
    tier: backend
```

**File: `hpa.yaml`**
```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: mekn-backend-hpa
  namespace: ahmedhemida
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mekn-backend
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 80
```

Apply the ExpressJS resources:
```sh
kubectl apply -f secrets.yaml
kubectl apply -f app-deployment.yaml
kubectl apply -f hpa.yaml
```

---

### 4. Configure Ingress
Expose the ExpressJS application to the outside world using Ingress.

**File: `ingress.yaml`**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mekn-ingress
  namespace: ahmedhemida
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: mekn-app.ahmedhemida.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: mongo-express
                port:
                  number: 8081
```

Apply the Ingress:
```sh
kubectl apply -f ingress.yaml
```

---

**Notes**:
- Ensure that the MongoDB credentials (`user` and `pass`) are correctly passed to the ExpressJS application for database connectivity.
- The ExpressJS application is secured with basic authentication using the provided credentials.

---

### 3. Verify the Application
- Access the ExpressJS application via `http://<node-ip>:32000`.
- Use the basic authentication credentials (`user` and `pass`) to log in.
- **Authentication**: ![Auth](auth.png)
- **Application Access**: ![Access](access.png)


### 4. Verify the Database
- Ensure that the MongoDB database is running and accessible from the ExpressJS application.
- **Database Overview**: ![database](db-overview.png)
- **Database Details**: ![database](database.png)

---

### **Project Components:**

1. **Kubernetes Cluster**:
   - The entire application runs inside a Kubernetes cluster.
   - The `mekn-app` namespace is used to isolate the application resources.

2. **Ingress**:
   - The Ingress controller routes external traffic to the ExpressJS application.
   - It uses TLS for secure communication (`mekn-secret`).
   - The host `mekn-app.ahmedhemida.com` is configured to route traffic to the `mongo-express` Service.

3. **ExpressJS Application**:
   - Deployed as a Kubernetes Deployment.
   - Exposed via a `NodePort` Service on port `32000`.
   - Uses **Secrets** for MongoDB credentials and basic authentication.
   - Includes **Liveness and Readiness Probes** for health checks.
   - Configured with **Horizontal Pod Autoscaler (HPA)** to automatically scale based on CPU usage.

4. **MongoDB Database**:
   - Deployed as a Kubernetes Deployment.
   - Exposed via a `ClusterIP` Service (internal only).
   - Uses **PersistentVolume (PV)** for data persistence.
   - Secured with authentication credentials stored in **Secrets**.

5. **Secrets**:
   - Stores sensitive information like MongoDB credentials (`user/pass`) and ExpressJS basic authentication (`user/pass`).
   - Accessed by both the ExpressJS and MongoDB deployments.

6. **Persistent Storage**:
   - MongoDB uses a PersistentVolume (`mongo-pv`) and PersistentVolumeClaim (`mongo-pvc`) to store data persistently.
   - Ensures data is retained even if the MongoDB pod is restarted.

7. **Health Checks**:
   - Both the ExpressJS and MongoDB deployments have **Liveness and Readiness Probes** to ensure the applications are running correctly.

8. **Horizontal Pod Autoscaler (HPA)**:
   - Automatically scales the ExpressJS application based on CPU usage (e.g., from 1 to 5 replicas).

---

### **Flow of Traffic**
1. A user accesses the application via `https://mekn-app.ahmedhemida.com`.
2. The Ingress controller routes the request to the `mongo-express` Service.
3. The `mongo-express` Service forwards the request to the ExpressJS application pod.
4. The ExpressJS application connects to the MongoDB database using the `mongodb-service` (ClusterIP).
5. MongoDB retrieves or stores data in the persistent volume.
