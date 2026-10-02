# Part 82: CI/CD Pipeline with GitHub Actions & Docker
## ขั้นตอนที่ 5641-5710: Automated Testing, Docker Build, K8s Deploy

---

## 82.1 CI/CD Overview

```
CI = Continuous Integration
  ทุก push → test → build → report

CD = Continuous Delivery/Deployment
  Delivery:   artifacts พร้อม deploy (manual trigger)
  Deployment: deploy อัตโนมัติเมื่อ test ผ่าน

Pipeline Stages:
  1. Code Check (lint, format, compile)
  2. Unit Tests
  3. Integration Tests (Testcontainers)
  4. Build Docker Image
  5. Security Scan (Trivy, OWASP)
  6. Push to Registry
  7. Deploy to Staging
  8. E2E Tests
  9. Deploy to Production (manual approval)
  
  Each stage: must pass before next stage runs

GitHub Actions Concepts:
  Workflow  = YAML file in .github/workflows/
  Event     = trigger (push, PR, schedule, manual)
  Job       = group of steps running on one runner
  Step      = single command or action
  Action    = reusable step (@actions/checkout etc.)
  Runner    = VM that runs jobs (ubuntu-latest)
```

---

## 82.2 Workflow สำหรับ Spring Boot

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]
  workflow_dispatch:   # manual trigger

env:
  JAVA_VERSION: '21'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ====== Job 1: Build & Test ======
  test:
    name: Build & Test
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'
      
      - name: Run tests
        run: mvn -B verify
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb
          SPRING_DATASOURCE_USERNAME: test
          SPRING_DATASOURCE_PASSWORD: test
          SPRING_REDIS_HOST: localhost
      
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: target/surefire-reports/
      
      - name: Publish test report
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: JUnit Tests
          path: target/surefire-reports/*.xml
          reporter: java-junit
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: target/site/jacoco/jacoco.xml

  # ====== Job 2: Security Scan ======
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: test
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload scan results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'

  # ====== Job 3: Build Docker Image ======
  build:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    needs: [test, security]
    if: github.event_name == 'push'
    
    permissions:
      contents: read
      packages: write
    
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'
      
      - name: Build JAR
        run: mvn -B package -DskipTests
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix=sha-
            type=semver,pattern={{version}}
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}
      
      - name: Build and push Docker image
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64

  # ====== Job 4: Deploy to Staging ======
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.example.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to K8s
        uses: azure/k8s-deploy@v4
        with:
          namespace: staging
          manifests: k8s/staging/
          images: |
            ${{ needs.build.outputs.image-tag }}

  # ====== Job 5: Deploy to Production (manual) ======
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://example.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to K8s Production
        uses: azure/k8s-deploy@v4
        with:
          namespace: production
          manifests: k8s/production/
          images: |
            ${{ needs.build.outputs.image-tag }}
      
      - name: Notify Slack
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Deployed ${{ github.repository }}@${{ github.sha }} to production"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 82.3 Dockerfile (Multi-stage)

```dockerfile
# Dockerfile

# Stage 1: Build
FROM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /build

# Cache dependencies separately
COPY pom.xml .
COPY .mvn .mvn
COPY mvnw .
RUN ./mvnw dependency:go-offline -q

# Build application
COPY src src
RUN ./mvnw package -DskipTests -q

# Extract layers for better caching
RUN java -Djarmode=layertools -jar target/*.jar extract

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-alpine AS runtime

# Security: non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

WORKDIR /app

# Copy layers (ordered by change frequency)
COPY --from=builder /build/dependencies .
COPY --from=builder /build/spring-boot-loader .
COPY --from=builder /build/snapshot-dependencies .
COPY --from=builder /build/application .

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD wget -q --spider http://localhost:8080/actuator/health || exit 1

EXPOSE 8080

# JVM flags for containers
ENV JAVA_OPTS="-XX:+UseContainerSupport \
               -XX:MaxRAMPercentage=75.0 \
               -XX:+UseZGC \
               -Djava.security.egd=file:/dev/./urandom"

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]
```

---

## 82.4 Kubernetes Manifests

```yaml
# k8s/production/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shop-api
  namespace: production
  labels:
    app: shop-api
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shop-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0   # zero downtime
  template:
    metadata:
      labels:
        app: shop-api
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/actuator/prometheus"
        prometheus.io/port: "8080"
    spec:
      serviceAccountName: shop-api-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
        - name: shop-api
          image: ghcr.io/myorg/shop-api:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
              name: http
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: production
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: host
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 30
            failureThreshold: 3
---
# k8s/production/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: shop-api
  namespace: production
spec:
  selector:
    app: shop-api
  ports:
    - port: 80
      targetPort: 8080
      name: http
---
# k8s/production/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: shop-api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: shop-api
  minReplicas: 3
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
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
```

---

## 82.5 GitHub Actions: Code Quality

```yaml
# .github/workflows/code-quality.yml
name: Code Quality

on:
  pull_request:
    branches: [ main, develop ]

jobs:
  quality:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # SonarQube needs full history
      
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'
      
      - name: Checkstyle
        run: mvn checkstyle:check
      
      - name: SpotBugs
        run: mvn spotbugs:check
      
      - name: PMD
        run: mvn pmd:check
      
      - name: SonarQube Scan
        run: mvn sonar:sonar
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
      
      - name: Dependency Check (OWASP)
        run: mvn dependency-check:check
        env:
          NVD_API_KEY: ${{ secrets.NVD_API_KEY }}
```

---

## สรุป Part 82

```
CI/CD Best Practices:

Pipeline Design:
  Fail fast: lint/compile first (seconds)
  Parallel jobs: test + security run at same time
  Gate: later stages need earlier to pass

Environment Strategy:
  develop branch → staging (auto)
  main branch    → production (manual approval)
  GitHub Environments → approval workflow

Docker Best Practices:
  Multi-stage build: smaller final image
  Layer ordering: dependencies before code (cache)
  Non-root user: security
  Health check: K8s knows when app is ready
  
  Image sizes:
    JDK image:  ~400MB
    JRE alpine: ~100MB
    Distroless:  ~50MB

K8s Deployment:
  rollingUpdate: zero downtime
  readinessProbe: traffic only when ready
  livenessProbe: restart if stuck
  HPA: auto-scale on CPU/memory

Security in CI:
  Never commit secrets (use GitHub Secrets)
  Scan dependencies (OWASP, Snyk, Trivy)
  SAST (SonarQube, SpotBugs)
  Container scanning (Trivy)
  SBOM generation

Monitoring:
  Notify Slack/Teams on deploy
  Compare metrics before/after
  Feature flags for gradual rollout
  Rollback plan: previous image tag
```

➡️ [Part 83: Monitoring & Observability](./Part-83-Observability.md)
