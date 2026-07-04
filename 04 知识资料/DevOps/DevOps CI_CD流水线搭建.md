---
title: DevOps实战：从零搭建企业级CI/CD流水线
date: 2026-07-02
tags: [devops, ci/cd, gitlab-ci, jenkins, kubernetes, docker]
category: 知识资料
source: https://mp.weixin.qq.com/s/FEB1zKrd5kGVlUtyi6AQSg
---

# DevOps 实战：从零搭建企业级 CI/CD 流水线

> 原文: [微信文章](https://mp.weixin.qq.com/s/FEB1zKrd5kGVlUtyi6AQSg)

---

## 一、流水线架构设计

三个关键组件：
1. **代码仓库**：GitLab/GitHub 触发 Webhook
2. **构建与测试**：Docker 多阶段构建 + 单元测试/代码扫描
3. **部署目标**：Kubernetes 集群（kubectl/Helm）

```
代码 Push → 自动触发 → 拉取代码 → 单元测试 → 镜像构建 → 推送仓库 → 部署 K8s → 健康检查 → 通知
```

---

## 二、GitLab CI 流水线（.gitlab-ci.yml）

```yaml
stages:
  - test
  - build
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA
  REGISTRY: registry.example.com

cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - .m2/repository/

# 阶段1：单元测试
单元测试:
  stage: test
  image: maven:3.8-openjdk-11
  script:
    - mvn clean test -DskipITs
  artifacts:
    reports:
      junit: target/surefire-reports/TEST-*.xml

# 阶段2：构建镜像（Docker-in-Docker）
构建镜像:
  stage: build
  image: docker:20.10
  services:
    - docker:20.10-dind
  script:
    - docker build -t $REGISTRY/myapp:$IMAGE_TAG .
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $REGISTRY
    - docker push $REGISTRY/myapp:$IMAGE_TAG
  only:
    - main
    - tags

# 阶段3：部署到 K8s
部署至K8s:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/myapp myapp=$REGISTRY/myapp:$IMAGE_TAG -n production
    - kubectl rollout status deployment/myapp -n production --timeout=5m
  environment:
    name: production
  only:
    - tags
```

> 关键点：
> - `$CI_COMMIT_SHORT_SHA` 作为镜像标签，**避免 latest 不可追溯**
> - 仅 `main` 分支和 `tags` 触发构建部署

---

## 三、Jenkins Pipeline（声明式）

```groovy
pipeline {
    agent any
    
    environment {
        REGISTRY = 'registry.example.com'
        IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
        K8S_NAMESPACE = 'production'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://gitlab.example.com/team/myapp.git'
            }
        }
        
        stage('Unit Test') {
            steps {
                sh 'mvn clean test -DskipITs'
            }
            post {
                always {
                    junit 'target/surefire-reports/TEST-*.xml'
                }
            }
        }
        
        stage('Build & Push Image') {
            steps {
                script {
                    docker.build("${REGISTRY}/myapp:${IMAGE_TAG}")
                    docker.withRegistry("https://${REGISTRY}", 'docker-credentials') {
                        docker.image("${REGISTRY}/myapp:${IMAGE_TAG}").push()
                    }
                }
            }
        }
        
        stage('Deploy to K8s') {
            when {
                tag pattern: "v\\d+\\.\\d+\\.\\d+", comparator: "REGEXP"
            }
            steps {
                sh """
                    kubectl set image deployment/myapp \
                      myapp=${REGISTRY}/myapp:${IMAGE_TAG} \
                      -n ${K8S_NAMESPACE}
                    kubectl rollout status deployment/myapp \
                      -n ${K8S_NAMESPACE} --timeout=5m
                """
            }
        }
    }
    
    post {
        success { emailext to: 'team@example.com', subject: "Build OK: ${env.JOB_NAME}" }
        failure { emailext to: 'team@example.com', subject: "Build FAILED: ${env.JOB_NAME}" }
    }
}
```

---

## 四、Docker 多阶段构建

```dockerfile
# 阶段1：编译
FROM maven:3.8-openjdk-11 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# 阶段2：运行（最小镜像）
FROM openjdk:11-jre-slim
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

好处：构建镜像包含完整 JDK/Maven，运行镜像只有 JRE，**镜像体积减少 60%+**。

---

## 五、K8s 部署策略

### 滚动更新（零停机）

```bash
kubectl set image deployment/myapp myapp=registry.example.com/myapp:v1.0.1
kubectl rollout status deployment/myapp --timeout=5m
```

### 蓝绿部署（通过 Service 切换）

```bash
# 部署绿色版本
kubectl apply -f deployment-green.yaml
# 验证后切换 Service
kubectl patch svc myapp -p '{"spec":{"selector":{"version":"green"}}}'
```

### 金丝雀发布（Istio/Flagger）

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
spec:
  service:
    port: 80
  analysis:
    interval: 1m
    threshold: 5
    maxWeight: 50
    stepWeight: 10
```

---

## 六、生产环境检查清单

- [ ] 镜像标签可追溯（不用 `latest`）
- [ ] 部署前通过集成测试
- [ ] 健康检查探针已配置
- [ ] 回滚方案已就绪
- [ ] 通知渠道已配置
- [ ] 密钥不入仓库（用 K8s Secret / Vault）
- [ ] 资源 limits/requests 已设置
- [ ] 日志采集已就绪

---

## 相关笔记

- [[K8s Deployment 实战指南]]
- [[K8s控制面故障应急响应复盘]]
