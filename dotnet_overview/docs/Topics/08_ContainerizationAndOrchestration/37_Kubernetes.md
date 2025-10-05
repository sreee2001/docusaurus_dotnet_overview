## 37. Kubernetes

### Short Introduction

Kubernetes is an open-source container orchestration platform that automates deployment, scaling, and management of containerized applications. For .NET applications, Kubernetes provides robust orchestration capabilities, enabling high availability, auto-scaling, and declarative configuration management across clusters.

### Official Definition

Kubernetes is a portable, extensible, open-source platform for managing containerized workloads and services that facilitates both declarative configuration and automation. It has a large, rapidly growing ecosystem with services, support, and tools widely available.

### Setup/Usage with .NET 8+ Code

**Basic Kubernetes Deployment for .NET API:**

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: hotel-management
---
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hotel-api
  namespace: hotel-management
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hotel-api
  template:
    metadata:
      labels:
        app: hotel-api
    spec:
      containers:
        - name: hotel-api
          image: hotel-management-api:latest
          ports:
            - containerPort: 8080
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: "Production"
            - name: ConnectionStrings__DefaultConnection
              valueFrom:
                secretKeyRef:
                  name: hotel-secrets
                  key: connection-string
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
---
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: hotel-api-service
  namespace: hotel-management
spec:
  selector:
    app: hotel-api
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
---
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hotel-api-ingress
  namespace: hotel-management
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
    - hosts:
        - api.hotelmanagement.com
      secretName: hotel-api-tls
  rules:
    - host: api.hotelmanagement.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hotel-api-service
                port:
                  number: 80
```

**ConfigMap and Secrets:**

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hotel-config
  namespace: hotel-management
data:
  appsettings.json: |
    {
      "Logging": {
        "LogLevel": {
          "Default": "Information",
          "Microsoft.AspNetCore": "Warning"
        }
      },
      "AllowedHosts": "*",
      "Redis": {
        "ConnectionString": "redis-service:6379"
      }
    }
---
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: hotel-secrets
  namespace: hotel-management
type: Opaque
data:
  connection-string: U2VydmVyPXNxbC1zZXJ2aWNlO0RhdGFiYXNlPUhvdGVsTWFuYWdlbWVudDtVc2VyIElkPXNhO1Bhc3N3b3JkPVlvdXJQYXNzd29yZDEyMyE7VHJ1c3RTZXJ2ZXJDZXJ0aWZpY2F0ZT10cnVl
```

### Use Cases

- **Microservices Orchestration**: Managing multiple interconnected services
- **Auto-scaling**: Horizontal and vertical scaling based on metrics
- **High Availability**: Multi-replica deployments with health checks
- **Rolling Updates**: Zero-downtime deployments
- **Service Discovery**: Internal communication between services
- **Configuration Management**: Centralized config and secrets management

### When to Use vs When Not to Use

**Use Kubernetes when:**

- Running multiple microservices
- Need high availability and scalability
- Managing complex deployment scenarios
- Operating in multi-environment setups
- Requiring advanced networking and storage
- Team has container orchestration expertise

**Consider alternatives when:**

- Simple single-service applications
- Small team with limited DevOps expertise
- Development or testing environments only
- Tight budget constraints (managed services cost)
- Windows-first applications with specific OS dependencies

### Market Alternatives & Pros/Cons

**Alternatives:**

- **Docker Swarm**: Simpler container orchestration
- **Amazon ECS/Fargate**: AWS-managed container services
- **Azure Container Instances**: Serverless containers
- **HashiCorp Nomad**: Simple, flexible orchestrator
- **OpenShift**: Enterprise Kubernetes platform

**Pros:**

- Industry standard with large ecosystem
- Powerful orchestration and scaling capabilities
- Declarative configuration management
- Strong community and vendor support
- Cloud-agnostic deployment
- Advanced networking and storage options

**Cons:**

- Steep learning curve and complexity
- Resource overhead for small applications
- Requires dedicated DevOps expertise
- Potential over-engineering for simple scenarios
- Configuration management complexity

### Complete Runnable Sample

**Complete Kubernetes Setup:**

```yaml
# k8s/hotel-management-complete.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: hotel-management
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: hotel-config
  namespace: hotel-management
data:
  appsettings.Production.json: |
    {
      "Logging": {
        "LogLevel": {
          "Default": "Information",
          "Microsoft.AspNetCore": "Warning"
        }
      },
      "ConnectionStrings": {
        "DefaultConnection": "Server=sql-service;Database=HotelManagement;User Id=sa;Password=YourPassword123!;TrustServerCertificate=true"
      },
      "Redis": {
        "ConnectionString": "redis-service:6379"
      },
      "JwtSettings": {
        "Issuer": "HotelManagement.Api",
        "Audience": "HotelManagement.Client"
      }
    }
---
apiVersion: v1
kind: Secret
metadata:
  name: hotel-secrets
  namespace: hotel-management
type: Opaque
stringData:
  jwt-secret: "your-super-secret-jwt-key-that-is-at-least-256-bits-long"
  sql-password: "YourPassword123!"
  redis-password: "YourRedisPassword123!"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hotel-api
  namespace: hotel-management
  labels:
    app: hotel-api
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: hotel-api
  template:
    metadata:
      labels:
        app: hotel-api
    spec:
      containers:
        - name: hotel-api
          image: hotel-management-api:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
              name: http
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: "Production"
            - name: ASPNETCORE_URLS
              value: "http://+:8080"
            - name: JwtSettings__SecretKey
              valueFrom:
                secretKeyRef:
                  name: hotel-secrets
                  key: jwt-secret
          volumeMounts:
            - name: config-volume
              mountPath: /app/appsettings.Production.json
              subPath: appsettings.Production.json
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
      volumes:
        - name: config-volume
          configMap:
            name: hotel-config
---
apiVersion: v1
kind: Service
metadata:
  name: hotel-api-service
  namespace: hotel-management
spec:
  selector:
    app: hotel-api
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sql-server
  namespace: hotel-management
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sql-server
  template:
    metadata:
      labels:
        app: sql-server
    spec:
      containers:
        - name: sql-server
          image: mcr.microsoft.com/mssql/server:2022-latest
          ports:
            - containerPort: 1433
          env:
            - name: ACCEPT_EULA
              value: "Y"
            - name: SA_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: hotel-secrets
                  key: sql-password
            - name: MSSQL_PID
              value: "Express"
          resources:
            requests:
              memory: "1Gi"
              cpu: "500m"
            limits:
              memory: "2Gi"
              cpu: "1000m"
          volumeMounts:
            - name: sql-data
              mountPath: /var/opt/mssql
      volumes:
        - name: sql-data
          persistentVolumeClaim:
            claimName: sql-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: sql-service
  namespace: hotel-management
spec:
  selector:
    app: sql-server
  ports:
    - port: 1433
      targetPort: 1433
  type: ClusterIP
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sql-pvc
  namespace: hotel-management
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hotel-api-hpa
  namespace: hotel-management
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hotel-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

**Deployment Commands:**

```bash
# Apply all configurations
kubectl apply -f k8s/hotel-management-complete.yaml

# Check deployment status
kubectl get deployments -n hotel-management
kubectl get pods -n hotel-management
kubectl get services -n hotel-management

# View logs
kubectl logs -f deployment/hotel-api -n hotel-management

# Scale deployment
kubectl scale deployment hotel-api --replicas=5 -n hotel-management

# Update deployment with new image
kubectl set image deployment/hotel-api hotel-api=hotel-management-api:1.1.0 -n hotel-management

# Port forward for local testing
kubectl port-forward service/hotel-api-service 8080:80 -n hotel-management

# Execute commands in pod
kubectl exec -it deployment/hotel-api -n hotel-management -- /bin/bash

# Clean up
kubectl delete namespace hotel-management
```
