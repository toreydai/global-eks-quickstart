# Demo01 — 准备实验环境与创建集群

## 实验简介

本实验将从零开始搭建一个完整的 EKS 集群环境，包括安装必要的 CLI 工具、创建托管节点组集群、验证集群连通性，并安装 Pod Identity Agent 插件为后续实验做准备。EKS 是 AWS 托管的 Kubernetes 服务，免去了控制平面运维负担，让团队专注于应用部署。

**实验目标：**
- 安装并验证 eksctl、kubectl v1.35、Helm 等核心工具
- 使用 eksctl 创建包含托管节点组的 EKS 1.35 集群
- 通过部署 nginx + NLB 验证集群网络连通性
- 安装 eks-pod-identity-agent 插件，为后续 IAM 集成做准备

**实验流程：**
1. 安装 eksctl、kubectl、Helm 等 CLI 工具
2. 创建 EKS 集群（2 节点 t3.medium）
3. 验证节点就绪状态
4. 部署 nginx 并通过 NLB 验证外部访问
5. 清理测试资源
6. 扩展节点到 3 个
7. 安装 eks-pod-identity-agent 插件并验证 DaemonSet

**预计 AI 执行时长：** 25-30 分钟（含集群创建等待 15 分钟）


## 前提条件

- **工具**：eksctl、kubectl v1.35、Helm、jq、Docker、git
- **权限**：EKS、EC2、IAM、CloudFormation 创建权限（建议 AdministratorAccess）
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

### 1. 安装工具（已预装则跳过）

```bash
# eksctl
if ! command -v eksctl &>/dev/null; then
  curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" | tar xz -C /tmp
  sudo mv /tmp/eksctl /usr/local/bin/
fi

# kubectl 1.35
if ! kubectl version --client 2>/dev/null | grep -q "v1.35"; then
  curl -Lo /tmp/kubectl "https://dl.k8s.io/release/v1.35.0/bin/linux/amd64/kubectl"
  sudo install -o root -g root -m 0755 /tmp/kubectl /usr/local/bin/kubectl
fi

# Helm
if ! command -v helm &>/dev/null; then
  curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
fi

# jq、Docker、git
sudo dnf install -y jq docker git

# 验证版本
eksctl version
kubectl version --client
helm version --short
```

**预期输出**：各工具版本号正常输出，kubectl 版本包含 `v1.35`

### 2. 创建 EKS 集群

```bash
eksctl create cluster \
  --name=${CLUSTER_NAME} \
  --node-type t3.medium \
  --nodes 2 \
  --managed \
  --alb-ingress-access \
  --region=${AWS_REGION} \
  --version 1.35 \
  --node-ami-family AmazonLinux2023
```

**预期输出**：最终输出 `EKS cluster "demo" in "us-east-1" region is ready`

> ⚠️ 集群创建约需 10-15 分钟。eksctl 会自动创建 CloudFormation Stack，删除集群请使用 `eksctl delete cluster`，不要手动删除 Stack。

> ⚠️ 如果集群 `demo` 已存在，修改 `CLUSTER_NAME=demo2` 后重新执行。若遇到 EIP 配额限制（`The maximum number of addresses has been reached`），需先释放未使用的 Elastic IP 或申请配额提升。

### 3. 验证节点就绪

```bash
kubectl get nodes
```

**预期输出**：2 个节点状态均为 `Ready`

### 4. 部署 nginx 并通过 NLB 对外访问

```bash
kubectl create deployment nginx --image=public.ecr.aws/nginx/nginx:latest --replicas=2
kubectl expose deployment nginx --port=80 --type=LoadBalancer
kubectl annotate svc nginx service.beta.kubernetes.io/aws-load-balancer-scheme=internet-facing
```

**预期输出**：`service/nginx exposed`

> ⚠️ NLB 默认创建为 internal（仅 VPC 内可访问）。必须添加 `aws-load-balancer-scheme: internet-facing` annotation 才能从公网访问。若已安装 AWS LB Controller，该 annotation 会触发 Controller 创建公网 NLB。

> ⚠️ 若集群已安装 AWS LB Controller 但 NLB 长时间未创建，检查 Controller Pod 日志是否报凭证错误。常见原因是 Pod Identity Association 缺失——需确认 `aws-load-balancer-controller` ServiceAccount 已关联正确的 IAM Role。

### 5. 等待 NLB 分配地址

```bash
# 轮询直到 EXTERNAL-IP 非空（约 2-3 分钟）
while true; do
  ELB=$(kubectl get svc nginx -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null)
  [[ -n "$ELB" ]] && break
  echo "等待 NLB 分配中..."
  sleep 15
done
echo "NLB 地址：${ELB}"
```

**预期输出**：输出一个 `*.elb.amazonaws.com` 格式的 hostname

> ⚠️ NLB DNS 解析 + target 健康检查共需约 3 分钟，ELB hostname 出现后仍需等待 target 健康。

### 6. 验证 nginx 可访问

```bash
# 再等 60s 让 target 健康检查通过
sleep 60
curl -s -o /dev/null -w "%{http_code}" http://${ELB}
```

**预期输出**：`200`

### 7. 清理 nginx 测试资源

```bash
kubectl delete deployment nginx
kubectl delete svc nginx
```

**预期输出**：`deployment.apps "nginx" deleted` 和 `service "nginx" deleted`

### 8. 扩展节点到 3 个

```bash
NODE_GROUP=$(eksctl get nodegroup --cluster ${CLUSTER_NAME} --region=${AWS_REGION} -o json | jq -r '.[].Name')
eksctl scale nodegroup \
  --cluster=${CLUSTER_NAME} \
  --nodes=3 \
  --name=${NODE_GROUP} \
  --nodes-max=3 \
  --region=${AWS_REGION}
```

**预期输出**：`nodegroup successfully scaled`

### 9. 等待第 3 个节点就绪

```bash
# 轮询直到 3 个节点全部 Ready
while true; do
  READY=$(kubectl get nodes --no-headers | grep -c " Ready ")
  echo "就绪节点数：${READY}/3"
  [[ "${READY}" == "3" ]] && break
  sleep 15
done
```

**预期输出**：`就绪节点数：3/3`

### 10. 安装 eks-pod-identity-agent 插件

```bash
aws eks create-addon \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name eks-pod-identity-agent \
  --region ${AWS_REGION}

# 轮询直到 ACTIVE
while true; do
  STATUS=$(aws eks describe-addon \
    --cluster-name ${CLUSTER_NAME} \
    --addon-name eks-pod-identity-agent \
    --region ${AWS_REGION} \
    --query 'addon.status' --output text)
  echo "插件状态：${STATUS}"
  [[ "${STATUS}" == "ACTIVE" ]] && break
  sleep 15
done
```

**预期输出**：`插件状态：ACTIVE`

### 11. 验证 DaemonSet 运行

```bash
kubectl get daemonset eks-pod-identity-agent -n kube-system
```

**预期输出**：`DESIRED`、`CURRENT`、`READY` 数值相同且等于节点数（3）

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 使用 eksctl 创建并扩展 EKS 托管节点组集群
- [ ] 通过 kubectl 验证节点状态和部署应用到集群
- [ ] 安装 EKS addon 插件并确认 DaemonSet 在所有节点运行

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl get nodes --no-headers \| wc -l \| tr -d ' '` | `3` |
| 2 | `kubectl get nodes --no-headers \| awk '{print $2}' \| sort -u` | `Ready` |
| 3 | `aws eks describe-addon --cluster-name demo --addon-name eks-pod-identity-agent --region us-east-1 --query 'addon.status' --output text` | `ACTIVE` |
| 4 | `kubectl get daemonset eks-pod-identity-agent -n kube-system -o jsonpath='{.status.numberReady}'` | `3` |

---

## 实验总结

本实验完成了 EKS 集群的从零搭建，包括工具安装、集群创建、网络连通性验证和节点扩展。核心概念包括托管节点组的弹性伸缩、LoadBalancer Service 暴露应用、以及 Pod Identity Agent 为后续 IAM 集成提供基础。下一个实验将在此集群上部署 ALB Ingress Controller，实现更精细的流量路由。

---

## 清理

```bash
# 清理测试资源（如未清理）
kubectl delete deployment nginx 2>/dev/null || true
kubectl delete svc nginx 2>/dev/null || true

# 删除集群（含所有节点组和 CloudFormation Stack，约 10 分钟）
eksctl delete cluster --name=${CLUSTER_NAME} --region=${AWS_REGION}
echo "清理完成"
```
