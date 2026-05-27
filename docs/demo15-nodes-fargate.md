# Demo15 — 管理计算节点与 Fargate

## 实验简介

EKS 支持多种计算模式：EC2 托管节点组提供完全控制的实例管理，Spot 实例可降低 70-90% 成本用于容错工作负载，而 Fargate 则实现完全无服务器的 Pod 运行环境。本实验通过实际操作对比这些计算选项的配置方式和调度行为。

### 实验目标

- 创建 Spot 托管节点组并通过 taint/toleration 实现工作负载定向调度
- 理解 Fargate Profile 的 namespace 选择器匹配机制
- 对比 EC2 节点与 Fargate 节点的 Pod 调度差异
- 掌握 Fargate Pod Execution Role 的信任策略配置

### 实验流程

1. 创建 Spot 托管节点组并部署容忍 Spot taint 的工作负载
2. 验证 Pod 调度到 Spot 节点后清理节点组
3. 创建 Fargate Pod Execution Role 和 Fargate Profile
4. 在匹配 namespace 中部署 Pod 并验证运行在 Fargate 节点

### 预计 AI 执行时长

6-8 分钟

## 前提条件

- **工具**：AWS CLI v2、eksctl、kubectl v1.35
- **权限**：AdministratorAccess
- **前提**：Demo01 已完成
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export CLUSTER_NAME=demo
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

---

## 步骤

## 第一部分：Spot 托管节点组

### 1. 创建 Spot 节点组配置文件

```bash
cat > /tmp/ng-spot.yaml << EOF
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: ${CLUSTER_NAME}
  region: ${AWS_REGION}
managedNodeGroups:
- name: ng-spot
  instanceTypes: ["t3.medium", "t3.large"]
  spot: true
  desiredCapacity: 2
  minSize: 1
  maxSize: 4
  taints:
    - key: spot
      value: "true"
      effect: NoSchedule
  labels:
    workload: spot
  amiFamily: AmazonLinux2023
EOF

echo "节点组配置文件已创建"
```

**预期输出**：打印"节点组配置文件已创建"

> ⚠️ 必须使用 `managedNodeGroups`（不是 `nodeGroups`），创建非托管节点组行为不同。eksctl 不支持 stdin 管道，配置文件必须先写入磁盘再用 `-f` 引用。

### 2. 创建 Spot 节点组

```bash
eksctl create nodegroup -f /tmp/ng-spot.yaml
```

**预期输出**：eksctl 输出节点组创建成功日志，约 3-5 分钟完成

```bash
kubectl get nodes -L workload
```

**预期输出**：新增 2 个节点，`WORKLOAD` 列显示 `spot`

### 3. 部署工作负载到 Spot 节点

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spot-workload
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spot-demo
  template:
    metadata:
      labels:
        app: spot-demo
    spec:
      tolerations:
      - key: spot
        value: "true"
        effect: NoSchedule
      nodeSelector:
        workload: spot
      containers:
      - name: nginx
        image: public.ecr.aws/docker/library/nginx:alpine
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
EOF

kubectl rollout status deployment/spot-workload --timeout=3m
kubectl get pods -l app=spot-demo -o wide
```

**预期输出**：所有 Pod Running，NODE 列显示 Spot 节点

### 4. 验证调度到 Spot 节点

```bash
kubectl get pods -l app=spot-demo \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u
kubectl get nodes -l workload=spot -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
```

**预期输出**：Pod 所在节点名均属于 `workload=spot` 节点列表

### 5. 清理 Spot 工作负载和节点组

```bash
kubectl delete deployment spot-workload

eksctl delete nodegroup \
  --cluster ${CLUSTER_NAME} \
  --name ng-spot \
  --region ${AWS_REGION}

echo "Spot 节点组已删除"
```

**预期输出**：打印"Spot 节点组已删除"

---

## 第二部分：Fargate Profile

### 6. 创建 Fargate Pod Execution Role

```bash
FARGATE_ROLE_ARN=$(aws iam create-role \
  --role-name AmazonEKSFargatePodExecutionRole-${CLUSTER_NAME} \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{"Effect":"Allow","Principal":{"Service":"eks-fargate-pods.amazonaws.com"},"Action":"sts:AssumeRole"}]
  }' \
  --query Role.Arn --output text)

aws iam attach-role-policy \
  --role-name AmazonEKSFargatePodExecutionRole-${CLUSTER_NAME} \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSFargatePodExecutionRolePolicy

echo "Fargate Role ARN: ${FARGATE_ROLE_ARN}"
```

**预期输出**：打印 Fargate Role ARN

### 7. 创建 Fargate Profile

```bash
aws eks create-fargate-profile \
  --cluster-name ${CLUSTER_NAME} \
  --fargate-profile-name fargate-demo \
  --pod-execution-role-arn ${FARGATE_ROLE_ARN} \
  --selectors '[{"namespace":"fargate-demo"}]' \
  --region ${AWS_REGION}
```

**预期输出**：命令成功，无报错

> ⚠️ 若报 `Misconfigured PodExecutionRole Trust Policy`，等待 10 秒后重试（IAM 角色需传播时间）。

```bash
until [ "$(aws eks describe-fargate-profile \
  --cluster-name ${CLUSTER_NAME} \
  --fargate-profile-name fargate-demo \
  --region ${AWS_REGION} \
  --query 'fargateProfile.status' --output text)" = "ACTIVE" ]; do
  echo "等待 Fargate Profile ACTIVE..."
  sleep 15
done
echo "Fargate Profile ACTIVE"
```

**预期输出**：`Fargate Profile ACTIVE`

### 8. 部署 Pod 到 Fargate

```bash
kubectl create namespace fargate-demo

kubectl create deployment fargate-nginx \
  --image=public.ecr.aws/docker/library/nginx:alpine \
  --namespace=fargate-demo

echo "等待 Fargate Pod 调度（冷启动约 3-5 分钟）..."
kubectl rollout status deployment/fargate-nginx -n fargate-demo --timeout=10m
```

**预期输出**：Deployment rollout 成功

> ⚠️ Fargate Pod 冷启动约 3-5 分钟（Fargate 需按需分配专用节点），比 EC2 节点慢，属正常现象。

### 9. 验证运行在 Fargate 节点

```bash
kubectl get pods -n fargate-demo -o wide
```

**预期输出**：Pod Running，NODE 列节点名以 `fargate-ip-` 开头

```bash
kubectl get node $(kubectl get pods -n fargate-demo \
  -o jsonpath='{.items[0].spec.nodeName}')
```

**预期输出**：节点标签包含 `eks.amazonaws.com/compute-type=fargate`

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 创建 Spot 托管节点组并通过 taint/toleration 定向调度工作负载
- [ ] 配置 Fargate Profile 使匹配 namespace 的 Pod 运行在无服务器节点上
- [ ] 区分 EC2 节点与 Fargate 节点的调度行为和适用场景

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl get pods -n fargate-demo --no-headers \| grep Running \| wc -l \| tr -d ' '` | `1` |
| 2 | `kubectl get pods -n fargate-demo -o jsonpath='{.items[0].spec.nodeName}' \| grep -c 'fargate-ip'` | `1` |
| 3 | `aws eks describe-fargate-profile --cluster-name demo --fargate-profile-name fargate-demo --region us-east-1 --query 'fargateProfile.status' --output text` | `ACTIVE` |

---

## 实验总结

本实验对比了 EKS 的三种计算模式：On-Demand 托管节点组提供稳定基线，Spot 节点组通过 taint 隔离实现成本优化，Fargate 提供完全无服务器的 Pod 运行环境免去节点管理。根据工作负载特性选择合适的计算模式是 EKS 成本优化和运维简化的核心策略。

---

## 清理

```bash
kubectl delete deployment fargate-nginx -n fargate-demo 2>/dev/null || true
kubectl delete namespace fargate-demo 2>/dev/null || true

aws eks delete-fargate-profile \
  --cluster-name ${CLUSTER_NAME} \
  --fargate-profile-name fargate-demo \
  --region ${AWS_REGION} 2>/dev/null || true

until [ "$(aws eks describe-fargate-profile \
  --cluster-name ${CLUSTER_NAME} \
  --fargate-profile-name fargate-demo \
  --region ${AWS_REGION} \
  --query 'fargateProfile.status' --output text 2>/dev/null)" != "DELETING" ]; do
  echo "等待 Fargate Profile 删除..."
  sleep 15
done

aws iam detach-role-policy \
  --role-name AmazonEKSFargatePodExecutionRole-${CLUSTER_NAME} \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSFargatePodExecutionRolePolicy 2>/dev/null || true
aws iam delete-role \
  --role-name AmazonEKSFargatePodExecutionRole-${CLUSTER_NAME} 2>/dev/null || true

echo "清理完成"
```
