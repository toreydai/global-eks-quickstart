# Demo10 — Cluster Autoscaler 节点自动伸缩

## 实验简介

本实验将完成「Cluster Autoscaler 节点自动伸缩」的全部操作，涵盖资源创建、配置、验证的完整流程。

**实验目标：**
- 掌握 Cluster Autoscaler 节点自动伸缩 的核心操作流程
- 理解相关组件的工作原理和配置要点
- 能够独立完成从部署到验证的完整闭环

**实验流程：**
1. 调整节点组 ASG 最大容量
2. 创建 Cluster Autoscaler IAM Policy
3. 创建 IAM Role 并建立 Pod Identity 关联
4. 部署 Cluster Autoscaler
5. 验证 CA Pod 运行
6. 触发扩容验证
7. 等待新节点加入
8. 缩容观察（可选）

**预计 AI 执行时长：** 3-5 分钟


## 前提条件

- **工具**：AWS CLI v2、eksctl、kubectl v1.35
- **权限**：AdministratorAccess
- **前提**：Demo01 已完成，eks-pod-identity-agent 插件已安装；若刚执行过 Demo09（Karpenter），必须先完成 Demo09 全部清理
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export CLUSTER_NAME=demo
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export CA_VERSION=v1.35.0
```

---

## 步骤

### 1. 调整节点组 ASG 最大容量

查询 ASG 名称并将 MaxSize 设为 5，为扩容预留空间：

```bash
ASG_NAME=$(aws autoscaling describe-auto-scaling-groups \
  --query "AutoScalingGroups[?contains(Tags[?Key=='eks:cluster-name'].Value,'${CLUSTER_NAME}')].AutoScalingGroupName" \
  --output text | head -1)

echo "ASG: ${ASG_NAME}"

aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name ${ASG_NAME} \
  --max-size 5

echo "MaxSize 已更新为 5"
```

**预期输出**：打印 ASG 名称，最后打印"MaxSize 已更新为 5"

### 2. 创建 Cluster Autoscaler IAM Policy

```bash
cat > /tmp/ca-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeScalingActivities",
        "autoscaling:DescribeTags",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeLaunchTemplateVersions",
        "ec2:DescribeInstanceTypes"
      ],
      "Resource": "*"
    }
  ]
}
EOF

CA_POLICY_ARN=$(aws iam create-policy \
  --policy-name ClusterAutoscalerPolicy-${CLUSTER_NAME} \
  --policy-document file:///tmp/ca-policy.json \
  --query Policy.Arn --output text)

echo "Policy ARN: ${CA_POLICY_ARN}"
```

**预期输出**：打印 Policy ARN

### 3. 创建 IAM Role 并建立 Pod Identity 关联

```bash
cat > /tmp/pod-trust.json << 'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"pods.eks.amazonaws.com"},"Action":["sts:AssumeRole","sts:TagSession"]}]}
EOF

CA_ROLE_ARN=$(aws iam create-role \
  --role-name ClusterAutoscalerRole-${CLUSTER_NAME} \
  --assume-role-policy-document file:///tmp/pod-trust.json \
  --query Role.Arn --output text)

aws iam attach-role-policy \
  --role-name ClusterAutoscalerRole-${CLUSTER_NAME} \
  --policy-arn ${CA_POLICY_ARN}

aws eks create-pod-identity-association \
  --cluster-name ${CLUSTER_NAME} \
  --namespace kube-system \
  --service-account cluster-autoscaler \
  --role-arn ${CA_ROLE_ARN} \
  --region ${AWS_REGION}

echo "Pod Identity 关联创建完成"
```

**预期输出**：打印"Pod Identity 关联创建完成"

### 4. 部署 Cluster Autoscaler

```bash
curl -Lo /tmp/cluster-autoscaler.yaml \
  https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

sed -i "s|<YOUR CLUSTER NAME>|${CLUSTER_NAME}|g" /tmp/cluster-autoscaler.yaml

kubectl apply -f /tmp/cluster-autoscaler.yaml

kubectl set image deployment/cluster-autoscaler \
  cluster-autoscaler=registry.k8s.io/autoscaling/cluster-autoscaler:${CA_VERSION} \
  -n kube-system

kubectl rollout status deployment/cluster-autoscaler -n kube-system --timeout=5m
echo "Cluster Autoscaler 已部署"
```

**预期输出**：`deployment "cluster-autoscaler" successfully rolled out`

### 5. 验证 CA Pod 运行

```bash
kubectl get pods -n kube-system -l app=cluster-autoscaler
```

**预期输出**：Pod 状态为 `Running`

### 6. 触发扩容验证

创建超出当前节点容量的 Deployment：

```bash
kubectl create deployment ca-test \
  --image=public.ecr.aws/docker/library/nginx:alpine \
  --replicas=10

kubectl set resources deployment ca-test \
  --requests=cpu=500m,memory=256Mi

echo "等待 Pod Pending 触发 CA 扩容..."
sleep 30
kubectl get pods | grep ca-test | grep -c Pending || echo "全部已调度"
kubectl get nodes
```

**预期输出**：节点数从 3 开始增加（CA 新增节点），所有 Pod 最终变为 Running

> ⚠️ CA 扩容需要 2-3 分钟：感知 Pending → 决策扩容 → ASG 新增实例 → 节点加入集群。耐心等待。

### 7. 等待新节点加入

```bash
until kubectl get pods -l app=ca-test --no-headers | grep -v Running | wc -l | grep -q "^0$"; do
  echo "仍有 Pod 未 Running，等待..."
  sleep 15
done
echo "所有 Pod 已 Running"
kubectl get nodes
```

**预期输出**：节点数增加，所有 Pod Running

### 8. 缩容观察（可选）

```bash
kubectl delete deployment ca-test
echo "已删除测试 Deployment，等待 CA 缩容（约 10 分钟）..."
kubectl get nodes -w
```

**预期输出**：约 10 分钟后节点数恢复到 3

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
| 1 | `kubectl get pods -n kube-system -l app=cluster-autoscaler --no-headers \| grep Running \| wc -l \| tr -d ' '` | `1` |
| 2 | `kubectl get deployment cluster-autoscaler -n kube-system -o jsonpath='{.status.readyReplicas}'` | `1` |
| 3 | `kubectl get configmap cluster-autoscaler-status -n kube-system -o jsonpath='{.data.status}' 2>/dev/null \| grep -c "autoscalerStatus" \| tr -d ' '` | `1` |
| 4 | `aws iam get-policy --policy-arn $(aws iam list-policies --query "Policies[?PolicyName=='ClusterAutoscalerPolicy-demo'].Arn" --output text) --query 'Policy.PolicyName' --output text` | `ClusterAutoscalerPolicy-demo` |

---

## 实验总结

本实验完成了「Cluster Autoscaler 节点自动伸缩」的全部操作，从资源创建到功能验证形成了完整闭环。通过动手实践，你已掌握了相关组件的核心配置方法和最佳实践。Demo11 将学习 CSI 存储驱动部署有状态应用。

---

## 清理

```bash
kubectl delete deployment ca-test 2>/dev/null || true
kubectl delete -f /tmp/cluster-autoscaler.yaml 2>/dev/null || true

aws eks delete-pod-identity-association \
  --cluster-name ${CLUSTER_NAME} \
  --association-id $(aws eks list-pod-identity-associations \
    --cluster-name ${CLUSTER_NAME} \
    --namespace kube-system \
    --service-account cluster-autoscaler \
    --query 'associations[0].associationId' --output text) \
  --region ${AWS_REGION} 2>/dev/null || true

aws iam detach-role-policy \
  --role-name ClusterAutoscalerRole-${CLUSTER_NAME} \
  --policy-arn $(aws iam list-policies --query "Policies[?PolicyName=='ClusterAutoscalerPolicy-demo'].Arn" --output text) 2>/dev/null || true

aws iam delete-role --role-name ClusterAutoscalerRole-${CLUSTER_NAME} 2>/dev/null || true
aws iam delete-policy --policy-arn $(aws iam list-policies --query "Policies[?PolicyName=='ClusterAutoscalerPolicy-demo'].Arn" --output text) 2>/dev/null || true

ASG_NAME=$(aws autoscaling describe-auto-scaling-groups \
  --query "AutoScalingGroups[?contains(Tags[?Key=='eks:cluster-name'].Value,'${CLUSTER_NAME}')].AutoScalingGroupName" \
  --output text | head -1)
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name ${ASG_NAME} \
  --max-size 3

echo "清理完成"
```
