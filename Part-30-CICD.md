# Part 30: CI/CD Pipelines
## ขั้นตอนที่ 2001-2070: Automation & Deployment

---

## 30.1 CI/CD Overview

```
CI/CD Pipeline:

Developer → Push Code → CI Trigger
                          ↓
                     Checkout Code
                          ↓
                     Unit Tests
                          ↓
                     Integration Tests
                          ↓
                     Build Artifact
                          ↓
                     Code Quality (SonarQube)
                          ↓
                     Security Scan (Snyk/OWASP)
                          ↓
                     Build Docker Image
                          ↓
                     Push to Registry
                          ↓
                     Deploy to Staging
                          ↓
                     E2E Tests
                          ↓
                     Manual Approval (Prod)
                          ↓
                     Deploy to Production
                          ↓
                     Health Check
                          ↓
                     Done ✅
```

---

## 30.2 GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  JAVA_VERSION: '21'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  
  # ====== Test ======
  test:
    name: Unit & Integration Tests
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK ${{ env.JAVA_VERSION }}
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'
      
      - name: Run tests
        run: mvn -B test
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb
          SPRING_DATASOURCE_USERNAME: testuser
          SPRING_DATASOURCE_PASSWORD: testpass
      
      - name: Upload test reports
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-reports
          path: target/surefire-reports/
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: target/site/jacoco/jacoco.xml
  
  # ====== Code Quality ======
  quality:
    name: Code Quality Analysis
    runs-on: ubuntu-latest
    needs: test
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # SonarQube needs full history
      
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'
      
      - name: SonarQube Analysis
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
        run: mvn -B sonar:sonar
               -Dsonar.projectKey=myapp
               -Dsonar.host.url=${SONAR_HOST_URL}
               -Dsonar.login=${SONAR_TOKEN}
  
  # ====== Security Scan ======
  security:
    name: Security Scanning
    runs-on: ubuntu-latest
    needs: test
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'myapp'
          path: '.'
          format: 'HTML'
          args: '--failBuildOnCVSS 9'
      
      - name: Upload security report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: security-report
          path: reports/
  
  # ====== Build & Push Docker ======
  build-push:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    needs: [test, quality, security]
    if: github.ref == 'refs/heads/main'
    
    permissions:
      contents: read
      packages: write
    
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Log in to Container Registry
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
            type=sha,format=long
            type=semver,pattern={{version}}
      
      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
  
  # ====== Deploy to Staging ======
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build-push
    environment: staging
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up kubectl
        uses: azure/setup-kubectl@v3
      
      - name: Configure kubectl
        run: |
          echo "${{ secrets.KUBE_CONFIG_STAGING }}" | base64 -d > kubeconfig
          export KUBECONFIG=./kubeconfig
      
      - name: Deploy to staging
        run: |
          export KUBECONFIG=./kubeconfig
          kubectl set image deployment/myapp \
            myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }} \
            -n staging
          kubectl rollout status deployment/myapp -n staging --timeout=5m
      
      - name: Run smoke tests
        run: |
          curl -f https://staging.example.com/actuator/health || exit 1
          echo "Smoke tests passed!"
  
  # ====== Deploy to Production ======
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment:
      name: production
      url: https://api.example.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to production
        run: |
          echo "${{ secrets.KUBE_CONFIG_PROD }}" | base64 -d > kubeconfig
          export KUBECONFIG=./kubeconfig
          kubectl set image deployment/myapp \
            myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${{ github.sha }} \
            -n production
          kubectl rollout status deployment/myapp -n production --timeout=10m
      
      - name: Verify deployment
        run: |
          for i in 1 2 3; do
            if curl -sf https://api.example.com/actuator/health; then
              echo "Production deployment verified!"
              exit 0
            fi
            sleep 10
          done
          echo "Health check failed!" && exit 1
      
      - name: Notify Slack on success
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "✅ Deployed ${{ github.sha }} to production",
              "channel": "#deployments"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
      
      - name: Rollback on failure
        if: failure()
        run: |
          export KUBECONFIG=./kubeconfig
          kubectl rollout undo deployment/myapp -n production
          echo "Rolled back to previous version"
```

---

## 30.3 GitLab CI/CD

```yaml
# .gitlab-ci.yml
stages:
  - test
  - quality
  - build
  - deploy-staging
  - deploy-production

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

cache:
  paths:
    - .m2/repository/

# ====== Test Stage ======
unit-tests:
  stage: test
  image: eclipse-temurin:21
  services:
    - postgres:16
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: testuser
    POSTGRES_PASSWORD: testpass
    SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/testdb
  script:
    - mvn -B test
  artifacts:
    reports:
      junit:
        - target/surefire-reports/TEST-*.xml
    paths:
      - target/site/jacoco/
    expire_in: 1 week

# ====== Quality Stage ======
sonarqube:
  stage: quality
  image: eclipse-temurin:21
  needs: [unit-tests]
  script:
    - mvn -B verify sonar:sonar
      -Dsonar.host.url=$SONAR_URL
      -Dsonar.login=$SONAR_TOKEN
  only:
    - main
    - develop

# ====== Build Stage ======
build-docker:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  needs: [unit-tests]
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $DOCKER_IMAGE .
    - docker push $DOCKER_IMAGE
    - docker tag $DOCKER_IMAGE $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - main

# ====== Deploy Staging ======
deploy-staging:
  stage: deploy-staging
  image: bitnami/kubectl:latest
  needs: [build-docker]
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - echo "$KUBE_CONFIG_STAGING" | base64 -d > kubeconfig
    - export KUBECONFIG=./kubeconfig
    - kubectl set image deployment/myapp myapp=$DOCKER_IMAGE -n staging
    - kubectl rollout status deployment/myapp -n staging --timeout=5m
  only:
    - main

# ====== Deploy Production ======
deploy-production:
  stage: deploy-production
  image: bitnami/kubectl:latest
  needs: [deploy-staging]
  environment:
    name: production
    url: https://api.example.com
  script:
    - echo "$KUBE_CONFIG_PROD" | base64 -d > kubeconfig
    - export KUBECONFIG=./kubeconfig
    - kubectl set image deployment/myapp myapp=$DOCKER_IMAGE -n production
    - kubectl rollout status deployment/myapp -n production --timeout=10m
  when: manual  # manual trigger for production
  only:
    - main
```

---

## 30.4 pom.xml for CI

```xml
<!-- pom.xml optimized for CI -->
<build>
  <plugins>
    <!-- Unit tests (fast) -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-surefire-plugin</artifactId>
      <configuration>
        <excludes>
          <exclude>**/*IT.java</exclude>
          <exclude>**/*IntegrationTest.java</exclude>
        </excludes>
        <reuseForks>true</reuseForks>
        <forkCount>2</forkCount>
      </configuration>
    </plugin>
    
    <!-- Integration tests -->
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-failsafe-plugin</artifactId>
      <executions>
        <execution>
          <goals>
            <goal>integration-test</goal>
            <goal>verify</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
    
    <!-- Code coverage -->
    <plugin>
      <groupId>org.jacoco</groupId>
      <artifactId>jacoco-maven-plugin</artifactId>
      <executions>
        <execution>
          <goals><goal>prepare-agent</goal></goals>
        </execution>
        <execution>
          <id>report</id>
          <phase>test</phase>
          <goals><goal>report</goal></goals>
        </execution>
        <execution>
          <id>check</id>
          <goals><goal>check</goal></goals>
          <configuration>
            <rules>
              <rule>
                <limits>
                  <limit>
                    <counter>LINE</counter>
                    <value>COVEREDRATIO</value>
                    <minimum>0.80</minimum>  <!-- 80% coverage required -->
                  </limit>
                </limits>
              </rule>
            </rules>
          </configuration>
        </execution>
      </executions>
    </plugin>
    
    <!-- Spring Boot Plugin -->
    <plugin>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-maven-plugin</artifactId>
      <configuration>
        <layers>
          <enabled>true</enabled>
        </layers>
        <image>
          <name>registry.example.com/${project.artifactId}:${project.version}</name>
          <env>
            <BP_JVM_VERSION>21</BP_JVM_VERSION>
          </env>
        </image>
      </configuration>
    </plugin>
  </plugins>
</build>
```

---

## 30.5 Branch Strategy

```
Git Flow:

main          ──●────────────────────────●── (production releases)
                 \                       /
release/1.0    ──●──●──●──●──●──────────
                              \
develop       ──●──●──●──●──●──●──●──●──  (integration)
                    |     |     |
feature/      ──●──●     ●──●  ●──●─(PR)

Trunk-Based Development (simpler, CI-friendly):

main          ──●──●──●──●──●──●──●──●──  (deploy on every commit)
               feature flags hide unfinished features

Branch naming:
  feature/JIRA-123-add-payment
  bugfix/JIRA-456-fix-login
  hotfix/JIRA-789-security-patch
  release/1.0.0

Commit message format (Conventional Commits):
  feat: add OAuth2 login
  fix: correct JWT token expiry calculation
  docs: update API documentation
  test: add unit tests for PaymentService
  refactor: extract email validation to utility class
  chore: upgrade Spring Boot to 3.2.0
  BREAKING CHANGE: rename API response field 'user_id' to 'userId'
```

---

## 30.6 Automated Release

```yaml
# .github/workflows/release.yml
name: Create Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'
      
      - name: Build
        run: mvn -B clean package -DskipTests
      
      - name: Generate changelog
        id: changelog
        run: |
          PREVIOUS_TAG=$(git describe --tags --abbrev=0 HEAD^ 2>/dev/null || echo "")
          if [ -n "$PREVIOUS_TAG" ]; then
            CHANGES=$(git log ${PREVIOUS_TAG}..HEAD --pretty=format:"- %s (%h)")
          else
            CHANGES=$(git log --pretty=format:"- %s (%h)" | head -20)
          fi
          echo "changes<<EOF" >> $GITHUB_OUTPUT
          echo "$CHANGES" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT
      
      - name: Create GitHub Release
        uses: ncipollo/release-action@v1
        with:
          artifacts: "target/*.jar"
          body: |
            ## Changes
            ${{ steps.changelog.outputs.changes }}
            
            ## Docker Image
            `docker pull ghcr.io/${{ github.repository }}:${{ github.ref_name }}`
          token: ${{ secrets.GITHUB_TOKEN }}
```

---

## สรุป Part 30

```
CI/CD Best Practices:

1. Fast feedback: Unit tests < 5 min
2. Fail fast: Test อยู่ต้น pipeline
3. Immutable artifacts: Build once, deploy anywhere
4. Environment parity: Dev ≈ Staging ≈ Prod
5. Blue/Green Deploy: Zero downtime
6. Feature flags: Decouple deploy from release
7. Automated rollback: Trigger on health check fail
8. Secrets management: ไม่ commit secrets เด็ดขาด

Pipeline Stages:
  build → test → quality → security → package → staging → prod
  
Metrics to track:
  - Deployment frequency (how often)
  - Lead time (commit to production)
  - Mean time to recovery (MTTR)
  - Change failure rate (rollbacks)
```

➡️ [Part 31: Kotlin DSL & Advanced Features](./Part-31-Kotlin-Advanced.md)
