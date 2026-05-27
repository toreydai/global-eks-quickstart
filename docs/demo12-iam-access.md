# Demo12 — 身份与访问控制

## 实验简介

EKS 提供多层身份与访问控制机制，包括 Pod Identity（让 Pod 安全获取 AWS 凭证）、Access Entry（精细控制 IAM 主体对集群的访问权限）以及 Secrets Store CSI Driver（将 Secrets Manager 中的密钥安全挂载到 Pod）。本实验将逐一实践这三种机制。

**实验目标：**
- 掌握 Pod Identity 为 Pod 分配 IAM 角色的配置方法
- 理解 EKS Access Entry 的权限边界与 namespace 级别隔离
- 能够独立完成 Secrets Store CSI Driver 集成 Secrets Manager 的端到端配置

**实验流程：**
1. 创建 Pod Identity 关联并验证 Pod 获取 S3 只读权限
2. 配置 Access Entry 绑定 View Policy，验证读写和跨 namespace 权限边界
3. 安装 Secrets Store CSI Driver，通过 IRSA 将 Secrets Manager 密钥挂载到 Pod

**预计 AI 执行时长：** 12-15 分钟

---

## 前提条件

- **工具**：AWS CLI v2、eksctl、kubectl v1.35、Helm
- **权限**：AdministratorAccess
- **前提**：Demo01 已完成，eks-pod-identity-agent 插件已安装
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export CLUSTER_NAME=demo
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

---

## 步骤

> ⚠️ 本实验安装 Secrets Store CSI DaemonSet，对节点 CPU 有额外需求。若前序 Demo 遗留工作负载导致节点资源不足，先清理：`kubectl delete ns troubleshoot-demo scheduling-demo hpa-demo 2>/dev/null; kubectl delete deployment workshop-nginx 2>/dev/null`

## 第一部分：Pod Identity 验证 S3 访问

### 1. 创建 Pod Identity IAM Role（S3 只读）

```bash
cat > /tmp/pod-trust.json << 'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"pods.eks.amazonaws.com"},"Action":["sts:AssumeRole","sts:TagSession"]}]}
EOF

S3_ROLE_ARN=$(aws iam create-role \
  --role-name EKS-S3-Echoer-Role \
  --assume-role-policy-document file:///tmp/pod-trust.json \
  --query Role.Arn --output text)

aws iam attach-role-policy \
  --role-name EKS-S3-Echoer-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

aws eks create-pod-identity-association \
  --cluster-name ${CLUSTER_NAME} \
  --namespace default \
  --service-account s3-echoer \
  --role-arn ${S3_ROLE_ARN} \
  --region ${AWS_REGION}

echo "Pod Identity 关联创建完成"
```

**预期输出**：打印"Pod Identity 关联创建完成"

### 2. 部署 Job 验证 Pod Identity 身份

```bash
kubectl create serviceaccount s3-echoer

kubectl apply -f - <<'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: s3-echoer
spec:
  template:
    spec:
      serviceAccountName: s3-echoer
      containers:
      - name: aws-cli
        image: public.ecr.aws/aws-cli/aws-cli:latest
        command: ["sh", "-c", "aws sts get-caller-identity && echo '---' && aws s3 ls"]
      restartPolicy: Never
EOF

kubectl wait --for=condition=Complete job/s3-echoer --timeout=2m
kubectl logs job/s3-echoer
```

**预期输出**：显示 Role ARN（含 `EKS-S3-Echoer-Role`），以及 S3 桶列表

---

## 第二部分：EKS Access Entry 权限控制

### 3. 创建演示 IAM Role 和 namespace

```bash
DEMO_ROLE_ARN=$(aws iam create-role \
  --role-name EKS-Access-Demo-Role \
  --assume-role-policy-document "{
    \"Version\":\"2012-10-17\",
    \"Statement\":[{
      \"Effect\":\"Allow\",
      \"Principal\":{\"AWS\":\"arn:aws:iam::${ACCOUNT_ID}:root\"},
      \"Action\":\"sts:AssumeRole\"
    }]}" \
  --query Role.Arn --output text)

kubectl create namespace access-demo
kubectl run test-pod --image=public.ecr.aws/docker/library/nginx:alpine \
  -n access-demo --restart=Never

echo "DEMO_ROLE_ARN: ${DEMO_ROLE_ARN}"
```

**预期输出**：打印 Demo Role ARN

### 4. 创建 Access Entry 并绑定 View Policy

```bash
aws eks create-access-entry \
  --cluster-name ${CLUSTER_NAME} \
  --principal-arn ${DEMO_ROLE_ARN} \
  --kubernetes-groups access-demo-viewers \
  --region ${AWS_REGION}

aws eks associate-access-policy \
  --cluster-name ${CLUSTER_NAME} \
  --principal-arn ${DEMO_ROLE_ARN} \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
  --access-scope type=namespace,namespaces=access-demo \
  --region ${AWS_REGION}

echo "Access Entry 已创建并绑定 View Policy（仅 access-demo namespace）"
```

**预期输出**：打印确认信息

### 5. 验证权限边界

```bash
# 为 Demo Role 添加 EKS DescribeCluster 权限（update-kubeconfig 需要）
aws iam put-role-policy \
  --role-name EKS-Access-Demo-Role \
  --policy-name EKSDescribeCluster \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"eks:DescribeCluster","Resource":"*"}]}'

# 使用 Demo Role 的临时凭证验证
CREDS=$(aws sts assume-role \
  --role-arn ${DEMO_ROLE_ARN} \
  --role-session-name access-demo-test \
  --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
  --output text)

export AWS_ACCESS_KEY_ID=$(echo $CREDS | awk '{print $1}')
export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | awk '{print $2}')
export AWS_SESSION_TOKEN=$(echo $CREDS | awk '{print $3}')

aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION} --role-arn ${DEMO_ROLE_ARN}

echo "=== 测试 1：读取 access-demo namespace（应成功）==="
kubectl get pods -n access-demo

echo "=== 测试 2：删除 Pod（应被拒绝）==="
kubectl delete pod test-pod -n access-demo 2>&1 || echo "已被拒绝"

echo "=== 测试 3：读取 default namespace（应被拒绝）==="
kubectl get pods -n default 2>&1 || echo "已被拒绝"

# 恢复原始凭证
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION}
```

**预期输出**：测试 1 成功返回 Pod 列表；测试 2 和测试 3 返回 Forbidden 错误

---

## 第三部分：Secrets Manager + Secrets Store CSI（选做）

### 6. 创建 Secrets Manager Secret

```bash
aws secretsmanager create-secret \
  --name test/demo/db-password \
  --secret-string '{"password":"SuperSecret123!","username":"admin"}' \
  --region ${AWS_REGION}

echo "Secret 已创建"
```

**预期输出**：打印"Secret 已创建"

### 7. 安装 Secrets Store CSI Driver

```bash
helm repo add secrets-store-csi-driver \
  https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system --set syncSecret.enabled=true

kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml

kubectl patch csidriver secrets-store.csi.k8s.io --type=merge \
  -p '{"spec":{"tokenRequests":[{"audience":"pods.eks.amazonaws.com"},{"audience":"sts.amazonaws.com"}]}}'

kubectl rollout status daemonset/csi-secrets-store-secrets-store-csi-driver -n kube-system --timeout=3m
echo "Secrets Store CSI 已安装"
```

**预期输出**：打印"Secrets Store CSI 已安装"

> ⚠️ CSIDriver 必须配置两个 audience（`pods.eks.amazonaws.com` 和 `sts.amazonaws.com`），缺少任一会导致 token 请求失败。

### 8. 创建 IRSA Role 和 SecretProviderClass

```bash
OIDC_ID=$(aws eks describe-cluster --name ${CLUSTER_NAME} \
  --query 'cluster.identity.oidc.issuer' --output text | sed 's|https://||')

SM_ROLE_ARN=$(aws iam create-role \
  --role-name EKS-SecretsManager-Role \
  --assume-role-policy-document "{
    \"Version\":\"2012-10-17\",
    \"Statement\":[{
      \"Effect\":\"Allow\",
      \"Principal\":{\"Federated\":\"arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_ID}\"},
      \"Action\":\"sts:AssumeRoleWithWebIdentity\",
      \"Condition\":{\"StringLike\":{\"${OIDC_ID}:sub\":\"system:serviceaccount:default:secrets-sa\"}}
    }]}" \
  --query Role.Arn --output text)

aws iam attach-role-policy \
  --role-name EKS-SecretsManager-Role \
  --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite

kubectl create serviceaccount secrets-sa \
  --annotations "eks.amazonaws.com/role-arn=${SM_ROLE_ARN}" 2>/dev/null || true

kubectl apply -f - <<EOF
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "test/demo/db-password"
        objectType: "secretsmanager"
    region: "${AWS_REGION}"
EOF

echo "SecretProviderClass 已创建"
```

**预期输出**：打印"SecretProviderClass 已创建"

> ⚠️ SecretProviderClass 必须显式指定 `region`，不会自动继承环境变量。

### 9. 部署 Pod 验证 Secret 挂载

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: secrets-pod
spec:
  serviceAccountName: secrets-sa
  containers:
  - name: app
    image: public.ecr.aws/docker/library/busybox:latest
    command: ["sleep", "3600"]
    volumeMounts:
    - name: secrets
      mountPath: /mnt/secrets-store
      readOnly: true
  volumes:
  - name: secrets
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: aws-secrets
EOF

kubectl wait --for=condition=Ready pod/secrets-pod --timeout=3m
kubectl exec secrets-pod -- cat /mnt/secrets-store/test_demo_db-password
```

**预期输出**：显示 JSON 格式的 secret 内容（含 `password` 和 `username`）

---

## 验收标准

完成本实验后，你应当能够：
- [ ] Pod 通过 Pod Identity 成功获取 AWS 临时凭证并列出 S3 桶
- [ ] Access Entry 绑定的 IAM Role 仅能读取指定 namespace 的资源，跨 namespace 和写操作被拒绝
- [ ] Secrets Manager 中的密钥通过 CSI Driver 成功挂载到 Pod 文件系统中

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl get job s3-echoer -o jsonpath='{.status.conditions[?(@.type=="Complete")].status}'` | `True` |
| 2 | `aws eks list-access-entries --cluster-name demo --region us-east-1 --query "accessEntries[?contains(@,'EKS-Access-Demo-Role')] \| length(@)" --output text` | `1` |
| 3 | `aws secretsmanager describe-secret --secret-id test/demo/db-password --region us-east-1 --query 'Name' --output text` | `test/demo/db-password` |

---

## 实验总结

本实验构建了 EKS 的三层身份与访问控制体系：Pod Identity 实现了 Pod 级别的 AWS 凭证分发，Access Entry 实现了 IAM 主体到 Kubernetes RBAC 的映射与 namespace 级隔离，Secrets Store CSI Driver 实现了外部密钥的安全注入。这三种机制共同构成了生产环境中"最小权限"原则的技术基础。下一步 Demo13 将在此安全基础上部署 Prometheus 与 Grafana 监控栈。

---

## 清理

```bash
kubectl delete pod secrets-pod s3-echoer test-pod 2>/dev/null || true
kubectl delete job s3-echoer 2>/dev/null || true
kubectl delete secretproviderclass aws-secrets 2>/dev/null || true
kubectl delete serviceaccount s3-echoer secrets-sa 2>/dev/null || true
kubectl delete namespace access-demo 2>/dev/null || true

helm uninstall csi-secrets-store -n kube-system 2>/dev/null || true
kubectl delete -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml 2>/dev/null || true

aws secretsmanager delete-secret --secret-id test/demo/db-password --force-delete-without-recovery --region ${AWS_REGION} 2>/dev/null || true

aws eks delete-access-entry \
  --cluster-name ${CLUSTER_NAME} \
  --principal-arn $(aws iam get-role --role-name EKS-Access-Demo-Role --query Role.Arn --output text) \
  --region ${AWS_REGION} 2>/dev/null || true

for ASSOC in $(aws eks list-pod-identity-associations --cluster-name ${CLUSTER_NAME} \
  --namespace default --service-account s3-echoer \
  --query 'associations[*].associationId' --output text 2>/dev/null); do
  aws eks delete-pod-identity-association --cluster-name ${CLUSTER_NAME} --association-id ${ASSOC} --region ${AWS_REGION}
done

aws iam detach-role-policy --role-name EKS-S3-Echoer-Role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess 2>/dev/null || true
aws iam detach-role-policy --role-name EKS-Access-Demo-Role --policy-arn arn:aws:iam::aws:policy/AmazonEKSViewPolicy 2>/dev/null || true
aws iam detach-role-policy --role-name EKS-SecretsManager-Role --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite 2>/dev/null || true
aws iam delete-role --role-name EKS-S3-Echoer-Role 2>/dev/null || true
aws iam delete-role --role-name EKS-Access-Demo-Role 2>/dev/null || true
aws iam delete-role --role-name EKS-SecretsManager-Role 2>/dev/null || true

echo "清理完成"
```
