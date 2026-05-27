# Demo03 — ECR 私有镜像与镜像发布

## 实验简介

本实验将完成「ECR 私有镜像与镜像发布」的全部操作，涵盖资源创建、配置、验证的完整流程。

**实验目标：**
- 掌握 ECR 私有镜像与镜像发布 的核心操作流程
- 理解相关组件的工作原理和配置要点
- 能够独立完成从部署到验证的完整闭环

**实验流程：**
1. 启动 Docker 并创建 ECR 私有仓库
2. 登录 ECR
3. 构建并推送 v1 镜像
4. 在 EKS 部署 v1 镜像
5. 构建并推送 v2 镜像（滚动更新）
6. 滚动更新到 v2
7. 查看 ECR 漏洞扫描结果
8. 配置生命周期策略
9. 配置跨区域复制（us-west-2）
10. 推送新镜像触发跨区域复制

**预计 AI 执行时长：** 6-8 分钟


## 前提条件

- **工具**：aws CLI、Docker、kubectl
- **权限**：ECR 完整权限、EKS 操作权限
- **前提**：Demo01 集群已创建，Docker 已安装并启动
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export CLUSTER_NAME=demo
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_REGISTRY=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
export REPO_NAME=workshop-nginx
```

---

## 步骤

### 1. 启动 Docker 并创建 ECR 私有仓库

```bash
sudo systemctl start docker
sudo usermod -aG docker $USER

aws ecr create-repository \
  --repository-name ${REPO_NAME} \
  --image-scanning-configuration scanOnPush=true \
  --region ${AWS_REGION}
```

**预期输出**：JSON 输出包含 `"repositoryUri"` 字段，格式为 `<account>.dkr.ecr.us-east-1.amazonaws.com/workshop-nginx`

### 2. 登录 ECR

```bash
aws ecr get-login-password --region ${AWS_REGION} \
  | sudo docker login --username AWS --password-stdin ${ECR_REGISTRY}
```

**预期输出**：`Login Succeeded`

> ⚠️ ECR 登录 token 有效期 12 小时，长时间操作需重新登录。

### 3. 构建并推送 v1 镜像

```bash
mkdir -p /tmp/workshop-nginx && cd /tmp/workshop-nginx

cat > Dockerfile << 'EOF'
FROM public.ecr.aws/nginx/nginx:latest
RUN echo "<h1>Workshop nginx v1</h1>" > /usr/share/nginx/html/index.html
EXPOSE 80
EOF

sudo docker build -t ${REPO_NAME}:v1 .
sudo docker tag ${REPO_NAME}:v1 ${ECR_REGISTRY}/${REPO_NAME}:v1
sudo docker push ${ECR_REGISTRY}/${REPO_NAME}:v1
```

**预期输出**：最终输出 `v1: digest: sha256:...` 表示推送成功

### 4. 在 EKS 部署 v1 镜像

```bash
kubectl create deployment workshop-nginx \
  --image=${ECR_REGISTRY}/${REPO_NAME}:v1 \
  --replicas=2

kubectl rollout status deployment/workshop-nginx --timeout=5m
kubectl get deployment workshop-nginx
```

**预期输出**：`deployment "workshop-nginx" successfully rolled out`，READY 为 `2/2`

### 5. 构建并推送 v2 镜像（滚动更新）

```bash
cd /tmp/workshop-nginx

cat > Dockerfile << 'EOF'
FROM public.ecr.aws/nginx/nginx:latest
RUN echo "<h1>Workshop nginx v2</h1>" > /usr/share/nginx/html/index.html
EXPOSE 80
EOF

sudo docker build -t ${REPO_NAME}:v2 .
sudo docker tag ${REPO_NAME}:v2 ${ECR_REGISTRY}/${REPO_NAME}:v2
sudo docker push ${ECR_REGISTRY}/${REPO_NAME}:v2
```

**预期输出**：最终输出 `v2: digest: sha256:...`

### 6. 滚动更新到 v2

```bash
kubectl set image deployment/workshop-nginx \
  workshop-nginx=${ECR_REGISTRY}/${REPO_NAME}:v2

kubectl rollout status deployment/workshop-nginx --timeout=5m
kubectl rollout history deployment/workshop-nginx
```

**预期输出**：`deployment "workshop-nginx" successfully rolled out`，rollout history 显示 2 个 revision

> ⚠️ 某些 kubectl 版本不支持 `kubectl rollout history --no-headers`，验证时改用 `grep "^[0-9]"` 过滤标题行。

### 7. 查看 ECR 漏洞扫描结果

```bash
# 等待扫描完成（约 30-60s）
sleep 60
aws ecr describe-image-scan-findings \
  --repository-name ${REPO_NAME} \
  --image-id imageTag=v1 \
  --region ${AWS_REGION} \
  --query 'imageScanFindings.findingSeverityCounts'
```

**预期输出**：JSON 对象，显示各严重级别漏洞计数（如 `{"MEDIUM": 2, "LOW": 5}`），或 `null`（无漏洞）

### 8. 配置生命周期策略

```bash
aws ecr put-lifecycle-policy \
  --repository-name ${REPO_NAME} \
  --region ${AWS_REGION} \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "保留最近5个tagged镜像",
        "selection": {
          "tagStatus": "tagged",
          "tagPrefixList": ["v"],
          "countType": "imageCountMoreThan",
          "countNumber": 5
        },
        "action": {"type": "expire"}
      },
      {
        "rulePriority": 2,
        "description": "清理untagged镜像",
        "selection": {
          "tagStatus": "untagged",
          "countType": "sinceImagePushed",
          "countUnit": "days",
          "countNumber": 1
        },
        "action": {"type": "expire"}
      }
    ]
  }'
echo "生命周期策略配置完成"
```

**预期输出**：`生命周期策略配置完成`

### 9. 配置跨区域复制（us-west-2）

```bash
aws ecr put-replication-configuration \
  --region ${AWS_REGION} \
  --replication-configuration '{
    "rules": [
      {
        "destinations": [
          {"region": "us-west-2", "registryId": "'"${ACCOUNT_ID}"'"}
        ],
        "repositoryFilters": [
          {"filter": "workshop-", "filterType": "PREFIX_MATCH"}
        ]
      }
    ]
  }'
echo "跨区域复制配置完成"
```

**预期输出**：`跨区域复制配置完成`

> ⚠️ 跨区域复制是 registry 级别配置，影响该账号下所有仓库。规则仅对规则建立后的**新推送**生效，已有镜像不会自动同步。清理时必须将复制规则清空。

### 10. 推送新镜像触发跨区域复制

```bash
# 推送 v2 到目标区域（触发复制）
sudo docker push ${ECR_REGISTRY}/${REPO_NAME}:v2

# 等待复制（约 2-3 分钟）
sleep 180

# 验证 us-west-2 是否有镜像
aws ecr describe-images \
  --repository-name ${REPO_NAME} \
  --region us-west-2 \
  --query 'imageDetails[*].imageTags[]' 2>/dev/null || echo "复制可能仍在进行中"
```

**预期输出**：`["v2"]` 或类似列表

### 11. 演示 Pull Through Cache（public.ecr.aws 前缀）

```bash
# 创建 pull-through cache 规则
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix "public-cache" \
  --upstream-registry-url "public.ecr.aws" \
  --region ${AWS_REGION} 2>/dev/null || echo "规则已存在，继续"

# 通过 pull-through cache 拉取镜像
sudo docker pull ${ECR_REGISTRY}/public-cache/docker/library/alpine:latest
echo "Pull Through Cache 拉取成功"
```

**预期输出**：`Pull Through Cache 拉取成功`

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 所有核心资源创建成功且状态正常
- [ ] 验证检查点中的所有命令返回预期结果
- [ ] 理解各组件之间的关联关系和配置要点

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws ecr describe-repositories --repository-names workshop-nginx --region us-east-1 --query 'repositories[0].repositoryName' --output text` | `workshop-nginx` |
| 2 | `aws ecr list-images --repository-name workshop-nginx --region us-east-1 --query 'imageIds[?imageTag!=null].imageTag' --output text \| tr '\t' '\n' \| sort \| tr '\n' ' ' \| tr -d ' '` | `v1v2` |
| 3 | `kubectl get deployment workshop-nginx -o jsonpath='{.status.readyReplicas}'` | `2` |
| 4 | `kubectl rollout history deployment/workshop-nginx \| grep "^[0-9]" \| wc -l \| tr -d ' '` | `2` |
| 5 | `aws ecr get-lifecycle-policy --repository-name workshop-nginx --region us-east-1 --query 'lifecyclePolicyText' --output text \| python3 -c "import sys,json; d=json.load(sys.stdin); print(len(d['rules']))"` | `2` |

---

## 实验总结

本实验完成了 ECR 私有镜像的完整生命周期管理，包括仓库创建、镜像构建推送、滚动更新、漏洞扫描、生命周期策略配置和跨区域复制。核心概念包括 ECR 登录认证机制、镜像标签管理与滚动更新策略、以及通过生命周期策略和跨区域复制实现镜像治理。下一个实验将学习 Kubernetes 健康检查机制和常见故障排查方法，帮助你在生产环境中快速定位和解决问题。

---

## 清理

```bash
# 清空跨区域复制规则（必须清空）
aws ecr put-replication-configuration \
  --region us-east-1 \
  --replication-configuration '{"rules":[]}'

# 删除 EKS 中的 Deployment
kubectl delete deployment workshop-nginx

# 删除 ECR 仓库（包含所有镜像）
aws ecr delete-repository \
  --repository-name ${REPO_NAME} \
  --force \
  --region us-east-1

# 清理本地 Docker 镜像
sudo docker rmi ${ECR_REGISTRY}/${REPO_NAME}:v1 ${ECR_REGISTRY}/${REPO_NAME}:v2 2>/dev/null || true

echo "清理完成"
```
