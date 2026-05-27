# Demo09 — Karpenter 节点自动伸缩

## 实验简介

Karpenter 是 AWS 开源的 Kubernetes 节点自动伸缩器，能够根据 Pending Pod 的需求快速启动最优实例类型，并在负载降低时自动合并和回收节点。本实验从零开始完成 Karpenter 的完整部署，包括 IAM 配置、Helm 安装、NodePool/EC2NodeClass 创建，并验证扩缩容行为。

**实验目标：**
- 掌握 Karpenter 的 IAM 权限体系（NodeRole + ControllerRole + Pod Identity）
- 理解 EC2NodeClass 和 NodePool 的配置逻辑
- 能够独立完成 Karpenter 部署并验证节点自动扩缩容

**实验流程：**
1. 为子网和安全组添加 Karpenter Discovery Tag
2. 部署 CloudFormation Stack 创建 KarpenterNodeRole
3. 创建 Controller Role 并配置 Pod Identity 关联
4. 通过 Helm 安装 Karpenter 并创建 NodePool/EC2NodeClass
5. 触发扩容验证新节点自动创建，缩容验证节点自动回收

**预计 AI 执行时长：** 10-12 分钟

---

## 前提条件

- **工具**：kubectl、Helm、aws CLI、eksctl
- **权限**：IAM 完整权限、EC2、EKS、CloudFormation 操作权限
- **前提**：Demo01 已完成，eks-pod-identity-agent 插件已安装
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export CLUSTER_NAME=demo
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_REGISTRY=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
export KARPENTER_VERSION=1.12.1
export KARPENTER_NAMESPACE=karpenter
```

---

## 步骤

### 1. 为子网添加 Karpenter Discovery Tag

```bash
# 获取集群所属 VPC
VPC_ID=$(aws eks describe-cluster \
  --name ${CLUSTER_NAME} \
  --query 'cluster.resourcesVpcConfig.vpcId' \
  --output text)
echo "VPC ID: ${VPC_ID}"

# 为所有子网打 discovery tag
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=${VPC_ID}" \
  --query 'Subnets[*].SubnetId' --output text)

for SUBNET_ID in ${SUBNET_IDS}; do
  aws ec2 create-tags \
    --resources ${SUBNET_ID} \
    --tags Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}
  echo "子网 ${SUBNET_ID} 已打标"
done
```

**预期输出**：每个子网均输出 `子网 subnet-xxx 已打标`

### 2. 为安全组添加 Karpenter Discovery Tag

```bash
# 获取节点安全组（EKS 自动创建的）
SG_IDS=$(aws ec2 describe-security-groups \
  --filters "Name=tag:aws:eks:cluster-name,Values=${CLUSTER_NAME}" \
  --query 'SecurityGroups[*].GroupId' --output text)

for SG_ID in ${SG_IDS}; do
  aws ec2 create-tags \
    --resources ${SG_ID} \
    --tags Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}
  echo "安全组 ${SG_ID} 已打标"
done
```

**预期输出**：每个安全组均输出 `安全组 sg-xxx 已打标`

> ⚠️ 子网和安全组必须都打 `karpenter.sh/discovery: <cluster-name>` tag，Karpenter NodeClass 才能发现并使用它们。

### 3. 部署 Karpenter CloudFormation Stack（创建 KarpenterNodeRole）

```bash
curl -fsSL https://raw.githubusercontent.com/aws/karpenter-provider-aws/v${KARPENTER_VERSION}/website/content/en/preview/getting-started/getting-started-with-karpenter/cloudformation.yaml \
  -o /tmp/karpenter-cf.yaml

aws cloudformation deploy \
  --stack-name karpenter-${CLUSTER_NAME} \
  --template-file /tmp/karpenter-cf.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides ClusterName=${CLUSTER_NAME}

# 轮询直到 CREATE_COMPLETE
while true; do
  STATUS=$(aws cloudformation describe-stacks \
    --stack-name karpenter-${CLUSTER_NAME} \
    --query 'Stacks[0].StackStatus' --output text)
  echo "CloudFormation 状态：${STATUS}"
  [[ "${STATUS}" == "CREATE_COMPLETE" || "${STATUS}" == "UPDATE_COMPLETE" ]] && break
  sleep 30
done
```

**预期输出**：`CloudFormation 状态：CREATE_COMPLETE`

> ⚠️ CloudFormation 只自动创建 `KarpenterNodeRole`，不创建 Controller Role。Controller Role 需手动创建。

### 4. 创建 Access Entry 授权 KarpenterNodeRole

```bash
NODE_ROLE_ARN=$(aws iam get-role --role-name KarpenterNodeRole-${CLUSTER_NAME} \
  --query 'Role.Arn' --output text)
echo "KarpenterNodeRole ARN: ${NODE_ROLE_ARN}"

# 创建 Access Entry（EC2_LINUX 类型）
aws eks create-access-entry \
  --cluster-name ${CLUSTER_NAME} \
  --principal-arn ${NODE_ROLE_ARN} \
  --type EC2_LINUX \
  --region ${AWS_REGION}
echo "Access Entry 创建完成"
```

**预期输出**：`Access Entry 创建完成`

### 5. 手动创建 Karpenter Controller Role（Pod Identity）

```bash
cat > /tmp/pod-trust.json << 'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"pods.eks.amazonaws.com"},"Action":["sts:AssumeRole","sts:TagSession"]}]}
EOF

aws iam create-role \
  --role-name KarpenterControllerRole-${CLUSTER_NAME} \
  --assume-role-policy-document file:///tmp/pod-trust.json
echo "Controller Role 已创建"
```

**预期输出**：`Controller Role 已创建`

### 6. 附加所有 KarpenterController*Policy 到 Controller Role

```bash
# 动态附加 CF 生成的所有 KarpenterController*Policy（含 ZonalShift，共 6 个）
# 注意：list-policies 有分页限制，直接从 CF Stack 获取更可靠
aws cloudformation list-stack-resources --stack-name karpenter-${CLUSTER_NAME} \
  --query "StackResourceSummaries[?ResourceType=='AWS::IAM::ManagedPolicy'].PhysicalResourceId" \
  --output text | tr '\t' '\n' | while read POLICY_ARN; do
  aws iam attach-role-policy \
    --role-name KarpenterControllerRole-${CLUSTER_NAME} \
    --policy-arn "${POLICY_ARN}"
  echo "已附加 Policy：${POLICY_ARN}"
done
```

**预期输出**：每个 Policy 均输出 `已附加 Policy：arn:aws:iam::...`

### 7. 创建 Karpenter Pod Identity 关联

```bash
CONTROLLER_ROLE_ARN=$(aws iam get-role \
  --role-name KarpenterControllerRole-${CLUSTER_NAME} \
  --query 'Role.Arn' --output text)

aws eks create-pod-identity-association \
  --cluster-name ${CLUSTER_NAME} \
  --namespace ${KARPENTER_NAMESPACE} \
  --service-account karpenter \
  --role-arn ${CONTROLLER_ROLE_ARN} \
  --region ${AWS_REGION}
echo "Karpenter Pod Identity 关联创建完成"
```

**预期输出**：`Karpenter Pod Identity 关联创建完成`

### 8. 安装 Karpenter（Helm）

```bash
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version ${KARPENTER_VERSION} \
  --namespace ${KARPENTER_NAMESPACE} --create-namespace \
  --set serviceAccount.create=true \
  --set serviceAccount.name=karpenter \
  --set settings.clusterName=${CLUSTER_NAME} \
  --set settings.interruptionQueue=${CLUSTER_NAME} \
  --set controller.image.digest="" \
  --wait
```

**预期输出**：`STATUS: deployed`

> ⚠️ Helm 安装必须加 `--set controller.image.digest=""`，否则会因 ECR push 后 digest 不匹配导致失败。

### 9. 验证 Karpenter Pod 运行

```bash
kubectl get pods -n ${KARPENTER_NAMESPACE}
```

**预期输出**：`karpenter-*` Pod 状态为 `Running`（2/2 或 1/1）

### 10. 创建 EC2NodeClass

```bash
kubectl apply -f - <<EOF
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiSelectorTerms:
  - alias: al2023@latest
  role: KarpenterNodeRole-${CLUSTER_NAME}
  subnetSelectorTerms:
  - tags:
      karpenter.sh/discovery: ${CLUSTER_NAME}
  securityGroupSelectorTerms:
  - tags:
      karpenter.sh/discovery: ${CLUSTER_NAME}
  tags:
    karpenter.sh/discovery: ${CLUSTER_NAME}
EOF
```

**预期输出**：`ec2nodeclass.karpenter.k8s.aws/default created`

### 11. 创建 NodePool

```bash
kubectl apply -f - <<'EOF'
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      requirements:
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["on-demand"]
      - key: node.kubernetes.io/instance-type
        operator: In
        values: ["t3.medium", "t3.large", "t3a.medium"]
  limits:
    cpu: "20"
    memory: 40Gi
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 30s
EOF
```

**预期输出**：`nodepool.karpenter.sh/default created`

### 12. 触发扩容（部署大量副本）

```bash
kubectl create deployment inflate \
  --image=public.ecr.aws/eks-distro/kubernetes/pause:3.7 \
  --replicas=1

kubectl set resources deployment inflate \
  --requests=cpu=500m

# 扩容到 20 副本，触发 Pending 进而触发 Karpenter 添加节点
kubectl scale deployment inflate --replicas=20

echo "等待 Karpenter 响应（观察新节点启动）..."
```

**预期输出**：`deployment.apps/inflate scaled`

### 13. 观察 Karpenter 自动添加节点

```bash
# 轮询直到新节点出现（约 1-3 分钟）
INITIAL_NODES=$(kubectl get nodes --no-headers | wc -l)
echo "初始节点数：${INITIAL_NODES}"

while true; do
  CURRENT_NODES=$(kubectl get nodes --no-headers | wc -l)
  echo "当前节点数：${CURRENT_NODES}（$(date +%H:%M:%S)）"
  [[ "${CURRENT_NODES}" -gt "${INITIAL_NODES}" ]] && break
  sleep 15
done

kubectl get nodes
kubectl get pods -l app=inflate --no-headers | grep -c Running
```

**预期输出**：节点数增加（新节点由 Karpenter 创建）；Running Pod 数量增加

### 14. 验证缩容 — Karpenter 自动回收节点

```bash
kubectl scale deployment inflate --replicas=0
echo "等待 Karpenter 回收空闲节点（约 90s，consolidateAfter=30s）..."

# 轮询直到节点数恢复到初始值
while true; do
  CURRENT_NODES=$(kubectl get nodes --no-headers | wc -l)
  echo "当前节点数：${CURRENT_NODES}（$(date +%H:%M:%S)）"
  [[ "${CURRENT_NODES}" -le "${INITIAL_NODES}" ]] && break
  sleep 15
done

kubectl get nodes
```

**预期输出**：节点数恢复到扩容前的数量

---

## 验收标准

完成本实验后，你应当能够：
- [ ] Karpenter Controller Pod 在 karpenter namespace 中正常 Running
- [ ] EC2NodeClass 和 NodePool 资源创建成功
- [ ] 部署大量 Pod 后 Karpenter 自动创建新节点并调度 Pending Pod
- [ ] 缩容后 Karpenter 自动回收空闲节点，节点数恢复到初始值

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl get pods -n karpenter --no-headers \| grep -c Running` | `1`（或更多，取决于副本数） |
| 2 | `kubectl get ec2nodeclass default -o jsonpath='{.metadata.name}'` | `default` |
| 3 | `kubectl get nodepool default -o jsonpath='{.metadata.name}'` | `default` |
| 4 | `aws iam get-role --role-name KarpenterControllerRole-demo --query 'Role.RoleName' --output text` | `KarpenterControllerRole-demo` |
| 5 | `aws eks list-access-entries --cluster-name demo --region us-east-1 --query 'accessEntries' --output text \| grep -c KarpenterNodeRole` | `1` |

---

## 实验总结

本实验从零完成了 Karpenter 的完整部署与验证：配置 Discovery Tag、创建 IAM 角色体系、通过 Helm 安装 Controller、定义 NodePool 和 EC2NodeClass，并通过扩缩容测试验证了 Karpenter 快速响应 Pending Pod 创建最优实例、空闲时自动合并回收节点的能力。Karpenter 是 AWS 推荐的节点伸缩方案，下一个实验将介绍传统的 Cluster Autoscaler 作为对比方案。

---

## 清理

```bash
# 缩容 inflate（如未缩容）
kubectl scale deployment inflate --replicas=0 2>/dev/null || true
kubectl delete deployment inflate 2>/dev/null || true

# 删除 NodePool 和 EC2NodeClass（Karpenter 会先驱逐节点）
kubectl delete nodepool default 2>/dev/null || true
kubectl delete ec2nodeclass default 2>/dev/null || true

# 等待 Karpenter 节点终止（约 2 分钟）
sleep 120

# 卸载 Karpenter
helm uninstall karpenter -n ${KARPENTER_NAMESPACE} 2>/dev/null || true
kubectl delete namespace ${KARPENTER_NAMESPACE} 2>/dev/null || true

# 删除 Pod Identity 关联
ASSOC_ID=$(aws eks list-pod-identity-associations \
  --cluster-name demo \
  --namespace karpenter \
  --service-account karpenter \
  --region us-east-1 \
  --query 'associations[0].associationId' --output text 2>/dev/null)
[[ -n "${ASSOC_ID}" && "${ASSOC_ID}" != "None" ]] && \
  aws eks delete-pod-identity-association \
    --cluster-name demo \
    --association-id ${ASSOC_ID} \
    --region us-east-1

# 删除 Access Entry
NODE_ROLE_ARN=$(aws iam get-role \
  --role-name KarpenterNodeRole-${CLUSTER_NAME} \
  --query 'Role.Arn' --output text 2>/dev/null)
[[ -n "${NODE_ROLE_ARN}" ]] && \
  aws eks delete-access-entry \
    --cluster-name demo \
    --principal-arn ${NODE_ROLE_ARN} \
    --region us-east-1 2>/dev/null || true

# 删除 IAM Controller Role（先 detach 所有 policies）
aws iam list-attached-role-policies \
  --role-name KarpenterControllerRole-${CLUSTER_NAME} \
  --query 'AttachedPolicies[*].PolicyArn' --output text 2>/dev/null | tr '\t' '\n' | while read ARN; do
  aws iam detach-role-policy \
    --role-name KarpenterControllerRole-${CLUSTER_NAME} \
    --policy-arn "${ARN}" 2>/dev/null || true
done
aws iam delete-role --role-name KarpenterControllerRole-${CLUSTER_NAME} 2>/dev/null || true

# 删除 CloudFormation Stack（含 KarpenterNodeRole 和 Policies）
aws cloudformation delete-stack --stack-name karpenter-${CLUSTER_NAME}

# 清理 Discovery Tags（子网）
VPC_ID=$(aws eks describe-cluster \
  --name ${CLUSTER_NAME} \
  --query 'cluster.resourcesVpcConfig.vpcId' \
  --output text 2>/dev/null)
if [[ -n "${VPC_ID}" ]]; then
  SUBNET_IDS=$(aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=${VPC_ID}" \
    --query 'Subnets[*].SubnetId' --output text)
  for SUBNET_ID in ${SUBNET_IDS}; do
    aws ec2 delete-tags \
      --resources ${SUBNET_ID} \
      --tags Key=karpenter.sh/discovery 2>/dev/null || true
  done
fi

echo "清理完成"
```
