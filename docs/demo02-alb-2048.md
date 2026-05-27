# Demo02 — 部署 2048 应用与 AWS Load Balancer Controller

## 实验简介

本实验将安装 AWS Load Balancer Controller（LBC）并通过 Ingress 资源将经典 2048 游戏暴露到公网。LBC 是 EKS 生态中最核心的入口组件，它监听 Kubernetes Ingress/Service 资源并自动创建 ALB/NLB，是生产环境流量入口的标准方案。

**实验目标：**
- 掌握通过 Pod Identity 为 LBC 配置 IAM 权限的完整流程
- 理解 Ingress 资源与 ALB 的映射关系
- 能够独立完成从 Helm 安装 LBC 到应用对外暴露的全链路

**实验流程：**
1. 创建 LBC 所需的 IAM Role 和 Policy
2. 配置 Pod Identity 关联
3. 通过 Helm 安装 AWS Load Balancer Controller
4. 部署 2048 游戏应用与 Ingress 资源
5. 验证 ALB 创建并访问应用

**预计 AI 执行时长：** 5-8 分钟

---

## 前提条件

- **工具**：eksctl、kubectl、Helm、aws CLI
- **权限**：IAM 创建角色和策略、EKS Pod Identity 操作权限
- **前提**：Demo01 已完成，eks-pod-identity-agent 插件已安装
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export CLUSTER_NAME=demo
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_REGISTRY=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
```

---

## 步骤

### 1. 创建 LBC IAM Role（Pod Identity 手动模式）

```bash
cat > /tmp/pod-trust.json << 'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"pods.eks.amazonaws.com"},"Action":["sts:AssumeRole","sts:TagSession"]}]}
EOF

ROLE_ARN=$(aws iam create-role \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --assume-role-policy-document file:///tmp/pod-trust.json \
  --query Role.Arn --output text)
echo "Role ARN: ${ROLE_ARN}"
```

**预期输出**：`Role ARN: arn:aws:iam::<account-id>:role/AmazonEKSLoadBalancerControllerRole`

> ⚠️ Pod Identity 必须手动创建 IAM Role，trust policy 指向 `pods.eks.amazonaws.com`。`eksctl create podidentityassociation --permission-policy-arns` 对 AWS 托管策略有 Cross-account 限制，不可用。

### 2. 下载并创建 LBC IAM Policy

```bash
curl -o /tmp/iam-policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

LBC_POLICY_ARN=$(aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file:///tmp/iam-policy.json \
  --query Policy.Arn --output text)
echo "Policy ARN: ${LBC_POLICY_ARN}"
```

**预期输出**：`Policy ARN: arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy`

### 3. 挂载 Policy 到 Role

```bash
aws iam attach-role-policy \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --policy-arn ${LBC_POLICY_ARN}
echo "Policy 挂载完成"
```

**预期输出**：`Policy 挂载完成`

### 4. 创建 Pod Identity 关联

```bash
aws eks create-pod-identity-association \
  --cluster-name ${CLUSTER_NAME} \
  --namespace kube-system \
  --service-account aws-load-balancer-controller \
  --role-arn ${ROLE_ARN} \
  --region ${AWS_REGION}
```

**预期输出**：JSON 输出包含 `"associationId"` 字段

### 5. 安装 AWS Load Balancer Controller（Helm）

```bash
helm repo add eks https://aws.github.io/eks-charts && helm repo update eks

helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=${CLUSTER_NAME} \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set enableShield=false \
  --set enableWaf=false \
  --set enableWafv2=false
```

**预期输出**：`STATUS: deployed`

### 6. 等待 LBC Deployment 就绪

```bash
kubectl rollout status deployment/aws-load-balancer-controller -n kube-system --timeout=5m
```

**预期输出**：`deployment "aws-load-balancer-controller" successfully rolled out`

> ⚠️ 如果是 `helm upgrade`（非首次安装），webhook TLS 证书可能失效，导致后续 apply Service/Ingress 时报 `x509: certificate signed by unknown authority`。解决方法：在 Step 6 完成后执行 `kubectl rollout restart deployment/aws-load-balancer-controller -n kube-system` 并等待就绪。

### 7. 创建 game-2048 namespace 并部署应用

```bash
kubectl create namespace game-2048

kubectl apply -n game-2048 -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-2048
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: app-2048
  replicas: 2
  template:
    metadata:
      labels:
        app.kubernetes.io/name: app-2048
    spec:
      containers:
      - image: public.ecr.aws/l6m2t8p7/docker-2048:latest
        imagePullPolicy: Always
        name: app-2048
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: service-2048
spec:
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
  type: NodePort
  selector:
    app.kubernetes.io/name: app-2048
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-2048
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service-2048
            port:
              number: 80
EOF
```

**预期输出**：`deployment.apps/deployment-2048 created`、`service/service-2048 created`、`ingress.networking.k8s.io/ingress-2048 created`

> ⚠️ Ingress annotation 必须同时设置 `scheme: internet-facing` 和 `target-type: ip`，缺一不可。

### 8. 等待 ALB 分配地址

```bash
# 轮询直到 ALB ADDRESS 非空（约 3-5 分钟）
while true; do
  ADDRESS=$(kubectl get ingress ingress-2048 -n game-2048 \
    -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null)
  [[ -n "$ADDRESS" ]] && break
  echo "等待 ALB 分配中..."
  sleep 15
done
echo "ALB 地址：${ADDRESS}"
```

**预期输出**：输出一个 `*.elb.amazonaws.com` 格式的 ALB hostname

### 9. 验证 2048 应用可访问

```bash
# 等待 target 健康（约 60s）
sleep 60
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://${ADDRESS})
echo "HTTP 状态码：${HTTP_CODE}"
curl -s http://${ADDRESS} | grep -o "<title>2048</title>"
```

**预期输出**：`HTTP 状态码：200` 和 `<title>2048</title>`

> ⚠️ ALB 从 provisioning 到 active 状态通常需要 2-3 分钟，DNS 传播还需额外时间。如果 `curl` 返回 `000`（无法解析），请等待 ALB 状态变为 active 后再重试：`aws elbv2 describe-load-balancers --query 'LoadBalancers[?contains(DNSName, \`k8s-game2048\`)].State.Code' --output text`

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
| 1 | `kubectl get deployment aws-load-balancer-controller -n kube-system -o jsonpath='{.status.readyReplicas}'` | `2` |
| 2 | `kubectl get deployment deployment-2048 -n game-2048 -o jsonpath='{.status.readyReplicas}'` | `2` |
| 3 | `kubectl get ingress ingress-2048 -n game-2048 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' \| grep -c elb.amazonaws.com` | `1` |
| 4 | `aws iam get-role --role-name AmazonEKSLoadBalancerControllerRole --query 'Role.RoleName' --output text` | `AmazonEKSLoadBalancerControllerRole` |

---

## 实验总结

本实验完成了 AWS Load Balancer Controller 的完整部署流程，包括 IAM Role/Policy 创建、Pod Identity 关联配置、Helm 安装 LBC，以及通过 Ingress 将 2048 游戏应用暴露到互联网。核心概念包括 Pod Identity 手动模式的信任策略配置、Ingress 注解（scheme 和 target-type）对 ALB 行为的控制。下一个实验将学习如何使用 ECR 私有仓库管理容器镜像，为生产环境的镜像发布流程打下基础。

---

## 清理

```bash
# 删除应用资源
kubectl delete namespace game-2048

# 删除 Helm 安装的 LBC
helm uninstall aws-load-balancer-controller -n kube-system

# 删除 Pod Identity 关联
ASSOC_ID=$(aws eks list-pod-identity-associations \
  --cluster-name demo \
  --namespace kube-system \
  --service-account aws-load-balancer-controller \
  --region us-east-1 \
  --query 'associations[0].associationId' --output text)
aws eks delete-pod-identity-association \
  --cluster-name demo \
  --association-id ${ASSOC_ID} \
  --region us-east-1

# 删除 IAM 资源
LBC_POLICY_ARN=$(aws iam list-policies --query "Policies[?PolicyName=='AWSLoadBalancerControllerIAMPolicy'].Arn" --output text)
aws iam detach-role-policy \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --policy-arn ${LBC_POLICY_ARN}
aws iam delete-policy --policy-arn ${LBC_POLICY_ARN}
aws iam delete-role --role-name AmazonEKSLoadBalancerControllerRole

echo "清理完成"
```
