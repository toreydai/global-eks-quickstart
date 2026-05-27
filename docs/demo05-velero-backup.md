# Demo05 — Velero 备份恢复

## 实验简介

灾难恢复是生产环境的关键能力。Velero 是 Kubernetes 生态中最流行的备份恢复工具，支持将集群资源和持久卷备份到对象存储（如 S3），并在需要时恢复。本实验将演示 Velero 的完整安装配置和备份恢复流程，模拟 namespace 级别的灾难恢复场景。

**实验目标：**
- 掌握 Velero 在 EKS 上的安装配置（含 Pod Identity 和 S3 存储）
- 理解 Velero 备份和恢复的工作流程与验证方法
- 能够独立完成 namespace 级别的备份和灾难恢复操作

**实验流程：**
1. 创建 S3 备份桶和 IAM 权限
2. 配置 Pod Identity 并通过 Helm 安装 Velero
3. 部署演示应用并执行备份
4. 模拟灾难（删除 namespace）并执行恢复
5. 验证恢复结果完整性

**预计 AI 执行时长：** 5-8 分钟

---

## 前提条件

- **工具**：kubectl、Helm、aws CLI
- **权限**：S3 创建权限、IAM 创建权限、EKS 操作权限
- **前提**：Demo01 已完成，eks-pod-identity-agent 插件已安装
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export CLUSTER_NAME=demo
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_REGISTRY=${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
export VELERO_BUCKET=test-velero-${ACCOUNT_ID}
export VELERO_VERSION=v1.18.0
export VELERO_NS=velero
```

---

## 步骤

### 1. 创建 S3 备份桶（启用 versioning）

```bash
aws s3api create-bucket \
  --bucket ${VELERO_BUCKET} \
  --region ${AWS_REGION} \
  --create-bucket-configuration LocationConstraint=${AWS_REGION} 2>/dev/null || \
aws s3api create-bucket \
  --bucket ${VELERO_BUCKET} \
  --region ${AWS_REGION}

aws s3api put-bucket-versioning \
  --bucket ${VELERO_BUCKET} \
  --versioning-configuration Status=Enabled

echo "S3 桶创建完成：${VELERO_BUCKET}"
```

**预期输出**：`S3 桶创建完成：test-velero-<account-id>`

> ⚠️ S3 桶启用了 versioning，清理时需先删除所有版本和删除标记，否则 `aws s3 rb` 会报 `BucketNotEmpty`。

### 2. 创建 Velero IAM Policy（限定具体 bucket）

```bash
cat > /tmp/velero-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeVolumes",
        "ec2:DescribeSnapshots",
        "ec2:CreateTags",
        "ec2:CreateVolume",
        "ec2:CreateSnapshot",
        "ec2:DeleteSnapshot"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:PutObject",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Resource": "arn:aws:s3:::${VELERO_BUCKET}/*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::${VELERO_BUCKET}"
    }
  ]
}
EOF

VELERO_POLICY_ARN=$(aws iam create-policy \
  --policy-name VeleroS3Policy \
  --policy-document file:///tmp/velero-policy.json \
  --query Policy.Arn --output text)
echo "Policy ARN: ${VELERO_POLICY_ARN}"
```

**预期输出**：`Policy ARN: arn:aws:iam::<account-id>:policy/VeleroS3Policy`

> ⚠️ S3 IAM Policy 必须自定义并限定具体 bucket，禁止使用 AmazonS3FullAccess（最小权限原则）。

### 3. 创建 Velero IAM Role（Pod Identity）

```bash
cat > /tmp/pod-trust.json << 'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"pods.eks.amazonaws.com"},"Action":["sts:AssumeRole","sts:TagSession"]}]}
EOF

VELERO_ROLE_ARN=$(aws iam create-role \
  --role-name VeleroRole \
  --assume-role-policy-document file:///tmp/pod-trust.json \
  --query Role.Arn --output text)

aws iam attach-role-policy \
  --role-name VeleroRole \
  --policy-arn ${VELERO_POLICY_ARN}

echo "Velero Role ARN: ${VELERO_ROLE_ARN}"
```

**预期输出**：`Velero Role ARN: arn:aws:iam::<account-id>:role/VeleroRole`

### 4. 创建 Pod Identity 关联

```bash
aws eks create-pod-identity-association \
  --cluster-name ${CLUSTER_NAME} \
  --namespace ${VELERO_NS} \
  --service-account velero-server \
  --role-arn ${VELERO_ROLE_ARN} \
  --region ${AWS_REGION}
echo "Pod Identity 关联创建完成"
```

**预期输出**：`Pod Identity 关联创建完成`

> ⚠️ Pod Identity SA 名称为 `velero-server`（与 Helm chart 默认 SA 名一致），不要自定义。

### 5. 安装 Velero（Helm）

```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts && helm repo update

helm upgrade --install velero vmware-tanzu/velero \
  --namespace ${VELERO_NS} --create-namespace \
  --set configuration.backupStorageLocation[0].provider=aws \
  --set configuration.backupStorageLocation[0].bucket=${VELERO_BUCKET} \
  --set configuration.backupStorageLocation[0].config.region=${AWS_REGION} \
  --set configuration.volumeSnapshotLocation[0].provider=aws \
  --set configuration.volumeSnapshotLocation[0].config.region=${AWS_REGION} \
  --set initContainers[0].name=velero-plugin-for-aws \
  --set initContainers[0].image=velero/velero-plugin-for-aws:v1.12.0 \
  --set initContainers[0].volumeMounts[0].mountPath=/target \
  --set initContainers[0].volumeMounts[0].name=plugins \
  --set serviceAccount.server.create=true \
  --set serviceAccount.server.name=velero-server
```

**预期输出**：`STATUS: deployed`

### 6. 等待 Velero Deployment 就绪

```bash
kubectl rollout status deployment/velero -n ${VELERO_NS} --timeout=5m
```

**预期输出**：`deployment "velero" successfully rolled out`

### 7. 验证 BackupStorageLocation 状态为 Available

```bash
# 等待 BSL 可用（约 30s）
sleep 30

# 轮询直到 PHASE=Available
while true; do
  PHASE=$(kubectl get backupstoragelocation default -n ${VELERO_NS} \
    -o jsonpath='{.status.phase}' 2>/dev/null)
  echo "BSL 状态：${PHASE}"
  [[ "${PHASE}" == "Available" ]] && break
  sleep 10
done
```

**预期输出**：`BSL 状态：Available`

### 8. 创建演示应用（nginx + ConfigMap）

```bash
kubectl create namespace velero-demo

kubectl apply -n velero-demo -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  env: "production"
  version: "1.0"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - name: nginx
        image: public.ecr.aws/nginx/nginx:latest
EOF

kubectl rollout status deployment/nginx-demo -n velero-demo --timeout=3m
```

**预期输出**：`deployment "nginx-demo" successfully rolled out`

### 9. 安装 Velero CLI 并执行备份

> ⚠️ Velero 服务端容器不包含 velero CLI，需在本地安装客户端工具。

```bash
# 安装 velero CLI
curl -sL https://github.com/vmware-tanzu/velero/releases/download/${VELERO_VERSION}/velero-${VELERO_VERSION}-linux-amd64.tar.gz -o /tmp/velero.tar.gz
tar -xzf /tmp/velero.tar.gz -C /tmp
sudo mv /tmp/velero-${VELERO_VERSION}-linux-amd64/velero /usr/local/bin/

velero backup create demo-backup \
  --include-namespaces velero-demo \
  --wait

velero backup describe demo-backup
```

**预期输出**：backup Phase 为 `Completed`

```bash
# 验证 S3 中有备份文件
aws s3 ls s3://${VELERO_BUCKET}/backups/demo-backup/ --recursive | head -5
```

**预期输出**：列出备份元数据文件（`.json` 后缀）

### 10. 模拟灾难 — 删除 namespace

```bash
kubectl delete namespace velero-demo
echo "模拟灾难：namespace velero-demo 已删除"
kubectl get namespace velero-demo 2>&1 || echo "namespace 已不存在（符合预期）"
```

**预期输出**：`Error from server (NotFound)` 或 `namespace 已不存在（符合预期）`

### 11. 执行恢复

```bash
velero restore create demo-restore \
  --from-backup demo-backup \
  --wait

velero restore describe demo-restore
```

**预期输出**：restore Phase 为 `Completed`

### 12. 验证恢复成功

```bash
kubectl rollout status deployment/nginx-demo -n velero-demo --timeout=3m

# 验证 ConfigMap 内容
kubectl get configmap app-config -n velero-demo -o jsonpath='{.data}'
```

**预期输出**：`{"env":"production","version":"1.0"}`

---

## 验收标准

完成本实验后，你应当能够：
- [ ] Velero BackupStorageLocation 状态为 Available
- [ ] demo-backup 备份状态为 Completed 且 S3 中存在备份文件
- [ ] 删除 velero-demo namespace 后通过恢复操作成功重建
- [ ] 恢复后 ConfigMap 数据和 Deployment 副本数与备份前一致

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl get backupstoragelocation default -n velero -o jsonpath='{.status.phase}'` | `Available` |
| 2 | `velero backup get demo-backup -o json 2>/dev/null \| python3 -c "import sys,json; print(json.load(sys.stdin)['status']['phase'])"` | `Completed` |
| 3 | `velero restore get demo-restore -o json 2>/dev/null \| python3 -c "import sys,json; print(json.load(sys.stdin)['status']['phase'])"` | `Completed` |
| 4 | `kubectl get configmap app-config -n velero-demo -o jsonpath='{.data.env}'` | `production` |
| 5 | `kubectl get configmap app-config -n velero-demo -o jsonpath='{.data.version}'` | `1.0` |
| 6 | `kubectl get deployment nginx-demo -n velero-demo -o jsonpath='{.status.readyReplicas}'` | `2` |

---

## 实验总结

本实验完成了「Velero 备份恢复」的全部操作，从资源创建到功能验证形成了完整闭环。通过动手实践，你已掌握了相关组件的核心配置方法和最佳实践。Demo06 将学习 Helm 包管理和 GitOps 发布流程。

---

## 清理

```bash
# 删除 Velero 备份/恢复记录
velero backup delete demo-backup --confirm 2>/dev/null || true
velero restore delete demo-restore --confirm 2>/dev/null || true

# 删除演示 namespace
kubectl delete namespace velero-demo 2>/dev/null || true

# 卸载 Velero
helm uninstall velero -n ${VELERO_NS}
kubectl delete namespace ${VELERO_NS} 2>/dev/null || true

# 删除 Pod Identity 关联
ASSOC_ID=$(aws eks list-pod-identity-associations \
  --cluster-name demo \
  --namespace velero \
  --service-account velero-server \
  --region us-east-1 \
  --query 'associations[0].associationId' --output text 2>/dev/null)
[[ -n "${ASSOC_ID}" && "${ASSOC_ID}" != "None" ]] && \
  aws eks delete-pod-identity-association \
    --cluster-name demo \
    --association-id ${ASSOC_ID} \
    --region us-east-1

# 删除 IAM 资源
VELERO_POLICY_ARN=$(aws iam list-policies --query "Policies[?PolicyName=='VeleroS3Policy'].Arn" --output text)
aws iam detach-role-policy --role-name VeleroRole --policy-arn ${VELERO_POLICY_ARN} 2>/dev/null || true
aws iam delete-policy --policy-arn ${VELERO_POLICY_ARN} 2>/dev/null || true
aws iam delete-role --role-name VeleroRole 2>/dev/null || true

# 删除 S3 桶（先清空版本和删除标记）
VELERO_BUCKET=test-velero-$(aws sts get-caller-identity --query Account --output text)

# 删除所有版本
VERSIONS=$(aws s3api list-object-versions --bucket ${VELERO_BUCKET} \
  --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}' --output json 2>/dev/null)
[[ -n "${VERSIONS}" && "${VERSIONS}" != "null" ]] && \
  aws s3api delete-objects --bucket ${VELERO_BUCKET} --delete "${VERSIONS}" 2>/dev/null || true

# 删除所有删除标记
MARKERS=$(aws s3api list-object-versions --bucket ${VELERO_BUCKET} \
  --query '{Objects: DeleteMarkers[].{Key:Key,VersionId:VersionId}}' --output json 2>/dev/null)
[[ -n "${MARKERS}" && "${MARKERS}" != "null" ]] && \
  aws s3api delete-objects --bucket ${VELERO_BUCKET} --delete "${MARKERS}" 2>/dev/null || true

aws s3 rb s3://${VELERO_BUCKET} 2>/dev/null || true

echo "清理完成"
```
