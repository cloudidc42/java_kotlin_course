# Part 29: Docker & Kubernetes
## ขั้นตอนที่ 1931-2000: Container & Orchestration

---

## 29.1 Docker Basics

```bash
# ====== Docker Installation ======
# Ubuntu
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# macOS
brew install --cask docker

# ====== Docker Commands ======
docker --version
docker info

# Images
docker pull openjdk:21-slim
docker images                    # list local images
docker rmi openjdk:21-slim       # remove image
docker image prune -a            # clean all unused

# Containers
docker run hello-world
docker run -d -p 8080:8080 --name myapp myapp:latest
docker run -it ubuntu /bin/bash  # interactive terminal

docker ps                        # running containers
docker ps -a                     # all containers
docker stop myapp
docker start myapp
docker restart myapp
docker rm myapp                  # remove stopped container
docker logs myapp                # view logs
docker logs -f myapp             # follow logs
docker exec -it myapp /bin/bash  # enter container
docker stats myapp               # resource usage

# Copy files
docker cp myapp:/app/logs/app.log ./local-log.log
docker cp ./config.yml myapp:/app/config.yml
```

---

## 29.2 Dockerfile

```dockerfile
# ====== Multi-stage Dockerfile for Spring Boot ======
# Stage 1: Build
FROM maven:3.9-eclipse-temurin-21-alpine AS builder

WORKDIR /app
COPY pom.xml .
# Download dependencies (cached unless pom.xml changes)
RUN mvn dependency:go-offline -B

COPY src ./src
RUN mvn clean package -DskipTests -B

# Extract layers for better caching
RUN java -Djarmode=layertools -jar target/*.jar extract

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-alpine

# Security: run as non-root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

WORKDIR /app

# Copy extracted layers (dependencies → infrequently changing last)
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=30s --retries=3 \
  CMD wget -q -O /dev/null http://localhost:8080/actuator/health || exit 1

EXPOSE 8080

ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "org.springframework.boot.loader.launch.JarLauncher"]
```

```dockerfile
# ====== Dockerfile for Kotlin/Gradle ======
FROM gradle:8-jdk21-alpine AS builder
WORKDIR /app
COPY build.gradle.kts settings.gradle.kts ./
COPY gradle ./gradle
RUN gradle dependencies --no-daemon
COPY src ./src
RUN gradle bootJar --no-daemon

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
# Build and run
docker build -t myapp:1.0.0 .
docker build -t myapp:1.0.0 --no-cache .
docker run -d \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  -e DB_URL=jdbc:postgresql://db:5432/myapp \
  -v /host/logs:/app/logs \
  --memory=512m \
  --cpus=1.0 \
  --name myapp \
  myapp:1.0.0

# Tag and push to registry
docker tag myapp:1.0.0 registry.example.com/myapp:1.0.0
docker push registry.example.com/myapp:1.0.0
```

---

## 29.3 Docker Compose

```yaml
# docker-compose.yml (development)
version: '3.8'

services:
  
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: builder  # use build stage for hot reload
    ports:
      - "8080:8080"
      - "5005:5005"  # debug port
    environment:
      SPRING_PROFILES_ACTIVE: dev
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/appdb
      SPRING_DATASOURCE_USERNAME: appuser
      SPRING_DATASOURCE_PASSWORD: apppass
      SPRING_REDIS_HOST: redis
      JAVA_TOOL_OPTIONS: "-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./src:/app/src  # hot reload in dev
    restart: unless-stopped
  
  db:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: apppass
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 10s
      timeout: 5s
      retries: 5
  
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --requirepass redispass --maxmemory 256mb
    healthcheck:
      test: ["CMD", "redis-cli", "--pass", "redispass", "ping"]
      interval: 10s
    volumes:
      - redis_data:/data
  
  pgadmin:
    image: dpage/pgadmin4:latest
    ports:
      - "5050:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
    depends_on: [db]

volumes:
  postgres_data:
  redis_data:

# Usage:
# docker compose up -d          # start
# docker compose down           # stop (keeps volumes)
# docker compose down -v        # stop + delete volumes
# docker compose logs -f app    # follow app logs
# docker compose exec app bash  # shell into app container
# docker compose build --no-cache app  # rebuild
```

---

## 29.4 Kubernetes Basics

```yaml
# ====== Pod ======
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
    version: "1.0.0"
spec:
  containers:
    - name: myapp
      image: myapp:1.0.0
      ports:
        - containerPort: 8080
      env:
        - name: SPRING_PROFILES_ACTIVE
          value: "k8s"
      resources:
        requests:
          memory: "256Mi"
          cpu: "250m"
        limits:
          memory: "512Mi"
          cpu: "500m"
      readinessProbe:
        httpGet:
          path: /actuator/health/readiness
          port: 8080
        initialDelaySeconds: 30
        periodSeconds: 10
      livenessProbe:
        httpGet:
          path: /actuator/health/liveness
          port: 8080
        initialDelaySeconds: 60
        periodSeconds: 15
```

```yaml
# ====== Deployment ======
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # zero-downtime
  template:
    metadata:
      labels:
        app: myapp
        version: "1.0.0"
    spec:
      containers:
        - name: myapp
          image: registry.example.com/myapp:1.0.0
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: myapp-config
            - secretRef:
                name: myapp-secrets
          resources:
            requests: { memory: "256Mi", cpu: "250m" }
            limits: { memory: "512Mi", cpu: "1000m" }
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8080 }
            initialDelaySeconds: 30
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
            initialDelaySeconds: 60
      imagePullSecrets:
        - name: registry-credentials
---
# ====== Service ======
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: production
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP  # internal only
---
# ====== Ingress ======
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts: [api.example.com]
      secretName: myapp-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-service
                port:
                  number: 80
```

```yaml
# ====== ConfigMap & Secret ======
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: production
data:
  SPRING_PROFILES_ACTIVE: "k8s"
  SERVER_PORT: "8080"
  DB_HOST: "postgres-service"
  DB_PORT: "5432"
  DB_NAME: "appdb"
---
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
  namespace: production
type: Opaque
stringData:
  DB_USERNAME: "appuser"
  DB_PASSWORD: "supersecret"
  JWT_SECRET: "myJwtSecretKey12345678901234567890"
```

---

## 29.5 Horizontal Pod Autoscaling

```yaml
# HPA based on CPU/Memory
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
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
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120
```

---

## 29.6 kubectl Commands

```bash
# ====== Context & Cluster ======
kubectl config get-contexts
kubectl config use-context production-cluster
kubectl config current-context

# ====== Namespaces ======
kubectl get namespaces
kubectl create namespace production
kubectl config set-context --current --namespace=production

# ====== Deployments ======
kubectl apply -f k8s/               # apply all YAML in directory
kubectl apply -f deployment.yml
kubectl get deployments
kubectl describe deployment myapp
kubectl rollout status deployment/myapp
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp          # rollback
kubectl rollout undo deployment/myapp --to-revision=2

# ====== Pods ======
kubectl get pods
kubectl get pods -w                  # watch
kubectl describe pod myapp-xyz-abc
kubectl logs myapp-xyz-abc
kubectl logs myapp-xyz-abc --previous  # crashed pod logs
kubectl logs -f myapp-xyz-abc          # follow
kubectl exec -it myapp-xyz-abc -- /bin/sh

# ====== Services ======
kubectl get services
kubectl port-forward service/myapp-service 8080:80  # local access

# ====== Scaling ======
kubectl scale deployment myapp --replicas=5

# ====== Rolling Update ======
kubectl set image deployment/myapp myapp=myapp:2.0.0
kubectl annotate deployment myapp \
  kubernetes.io/change-cause="Upgrade to v2.0.0"

# ====== Debug ======
kubectl top nodes
kubectl top pods
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl describe pod myapp-xyz-abc | grep -A5 Events

# ====== Secrets ======
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=secret123
kubectl get secret db-secret -o yaml

# ====== Cleanup ======
kubectl delete deployment myapp
kubectl delete service myapp-service
kubectl delete -f k8s/
```

---

## 29.7 Spring Boot Actuator for K8s

```yaml
# application-k8s.yml
management:
  endpoint:
    health:
      probes:
        enabled: true  # /health/liveness and /health/readiness
      show-details: always
  endpoints:
    web:
      exposure:
        include: "health,info,metrics,prometheus"
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true

server:
  shutdown: graceful  # finish in-flight requests before shutdown

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

```java
// Custom Readiness Indicator
import org.springframework.boot.availability.*;
import org.springframework.stereotype.*;

@Component
public class DatabaseReadinessIndicator {
    
    private final javax.sql.DataSource dataSource;
    private final ApplicationEventPublisher eventPublisher;
    
    public DatabaseReadinessIndicator(javax.sql.DataSource dataSource,
                                      ApplicationEventPublisher eventPublisher) {
        this.dataSource = dataSource;
        this.eventPublisher = eventPublisher;
    }
    
    @org.springframework.scheduling.annotation.Scheduled(fixedRate = 30000)
    public void checkDatabase() {
        try (var conn = dataSource.getConnection()) {
            conn.createStatement().execute("SELECT 1");
            // DB is ready
            eventPublisher.publishEvent(
                new AvailabilityChangeEvent<>(this, ReadinessState.ACCEPTING_TRAFFIC)
            );
        } catch (Exception e) {
            // DB not ready
            eventPublisher.publishEvent(
                new AvailabilityChangeEvent<>(this, ReadinessState.REFUSING_TRAFFIC)
            );
        }
    }
}
```

---

## 29.8 Helm Chart

```yaml
# charts/myapp/Chart.yaml
apiVersion: v2
name: myapp
description: My Spring Boot Application
type: application
version: 0.1.0
appVersion: "1.0.0"
```

```yaml
# charts/myapp/values.yaml
replicaCount: 2

image:
  repository: registry.example.com/myapp
  pullPolicy: IfNotPresent
  tag: ""

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  host: api.example.com
  tls: true

resources:
  requests:
    memory: 256Mi
    cpu: 250m
  limits:
    memory: 512Mi
    cpu: 1000m

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

env:
  SPRING_PROFILES_ACTIVE: k8s
  DB_HOST: postgres-service
```

```yaml
# charts/myapp/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels: {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels: {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels: {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          env:
            {{- range $k, $v := .Values.env }}
            - name: {{ $k }}
              value: {{ $v | quote }}
            {{- end }}
          resources: {{- toYaml .Values.resources | nindent 12 }}
```

```bash
# Helm commands
helm create myapp                          # scaffold chart
helm install myapp ./charts/myapp          # install
helm install myapp ./charts/myapp -f prod-values.yml
helm upgrade myapp ./charts/myapp          # upgrade
helm rollback myapp 1                      # rollback to revision 1
helm list                                  # list releases
helm status myapp
helm uninstall myapp
helm template myapp ./charts/myapp         # dry-run (show YAML)
```

---

## สรุป Part 29

| Tool | ใช้สำหรับ |
|------|---------|
| Docker | Build & run containers |
| Docker Compose | Multi-container local dev |
| Kubernetes | Container orchestration |
| kubectl | K8s command line |
| Helm | K8s package manager |

**Production Checklist:**
- ✅ Multi-stage Dockerfile (minimize image size)
- ✅ Non-root user
- ✅ Resource limits (requests & limits)
- ✅ Readiness & liveness probes
- ✅ Graceful shutdown
- ✅ Horizontal Pod Autoscaling
- ✅ Secrets as K8s Secrets (not ConfigMaps)
- ✅ Rolling updates (maxUnavailable=0)

➡️ [Part 30: CI/CD Pipelines](./Part-30-CICD.md)
