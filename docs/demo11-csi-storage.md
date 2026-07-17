# Demo11 — 使用 CSI 部署有状态应用（EBS、EFS 与 S3）

## 实验简介

Kubernetes 原生不提供持久化存储能力，需要通过 Container Storage Interface（CSI）驱动将云存储服务集成到集群中。本实验在 EKS 集群上安装三种 AWS CSI 驱动（EBS、EFS、S3 Mountpoint），分别演示块存储、共享文件存储和对象存储的挂载方式，帮助理解不同存储类型的适用场景。

**实验目标：**
- 安装 EBS CSI Driver 并通过 StorageClass 动态创建 gp3 卷
- 安装 EFS CSI Driver 并实现多 Pod 共享读写同一文件系统
- 安装 S3 Mountpoint CSI Driver 并将 S3 桶挂载为 Pod 文件系统
- 理解 Pod Identity 与节点实例角色两种 CSI 权限模型的区别

**实验流程：**
1. 创建 EBS CSI IAM Role 并安装 addon，部署 Pod 验证块存储写入
2. 创建 EFS CSI IAM Role 并安装 addon，创建 EFS 文件系统和挂载目标
3. 部署两个 Pod 验证 EFS 共享读写
4. 创建 S3 桶并安装 S3 CSI addon，部署 Pod 验证对象存储挂载

**预计 AI 执行时长：** 8-10 分钟


## 前提条件

- **工具**：AWS CLI v2、eksctl、kubectl v1.35
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

## 第一部分：EBS CSI Driver

### 1. 创建 EBS CSI Driver IAM Role

```bash
cat > /tmp/pod-trust.json << 'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"pods.eks.amazonaws.com"},"Action":["sts:AssumeRole","sts:TagSession"]}]}
EOF

EBS_ROLE_ARN=$(aws iam create-role \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --assume-role-policy-document file:///tmp/pod-trust.json \
  --query Role.Arn --output text)

aws iam attach-role-policy \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy

echo "EBS CSI Role ARN: ${EBS_ROLE_ARN}"
```

**预期输出**：打印 Role ARN

### 2. 安装 EBS CSI Driver addon

```bash
aws eks create-addon \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name aws-ebs-csi-driver \
  --pod-identity-associations "serviceAccount=ebs-csi-controller-sa,roleArn=${EBS_ROLE_ARN}" \
  --region ${AWS_REGION}

until [ "$(aws eks describe-addon --cluster-name ${CLUSTER_NAME} --addon-name aws-ebs-csi-driver \
  --region ${AWS_REGION} --query 'addon.status' --output text)" = "ACTIVE" ]; do
  echo "等待 EBS CSI addon ACTIVE..."
  sleep 15
done
echo "EBS CSI addon ACTIVE"
```

**预期输出**：`EBS CSI addon ACTIVE`

### 3. 创建 StorageClass 和 PVC

```bash
kubectl apply -f - <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 1Gi
EOF

echo "StorageClass 和 PVC 已创建"
```

**预期输出**：打印"StorageClass 和 PVC 已创建"

### 4. 部署写入 Pod 验证 EBS

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: ebs-writer
spec:
  containers:
  - name: writer
    image: public.ecr.aws/docker/library/busybox:latest
    command: ["sh", "-c", "while true; do date >> /data/timestamps.txt; sleep 5; done"]
    volumeMounts:
    - name: ebs-vol
      mountPath: /data
  volumes:
  - name: ebs-vol
    persistentVolumeClaim:
      claimName: ebs-claim
EOF

kubectl wait --for=condition=Ready pod/ebs-writer --timeout=3m
sleep 15
kubectl exec ebs-writer -- tail -5 /data/timestamps.txt
```

**预期输出**：显示最近 5 行时间戳（EBS 持续写入成功）

---

## 第二部分：EFS CSI Driver

### 5. 创建 EFS CSI Driver IAM Role

```bash
EFS_ROLE_ARN=$(aws iam create-role \
  --role-name AmazonEKS_EFS_CSI_DriverRole \
  --assume-role-policy-document file:///tmp/pod-trust.json \
  --query Role.Arn --output text)

aws iam attach-role-policy \
  --role-name AmazonEKS_EFS_CSI_DriverRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEFSCSIDriverPolicy

echo "EFS CSI Role ARN: ${EFS_ROLE_ARN}"
```

**预期输出**：打印 Role ARN

### 6. 添加节点 EFS 访问权限

```bash
cat > /tmp/efs-node-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "elasticfilesystem:DescribeMountTargets",
      "elasticfilesystem:DescribeFileSystems",
      "elasticfilesystem:ClientMount",
      "elasticfilesystem:ClientWrite"
    ],
    "Resource": "*"
  }]
}
EOF

NODE_ROLE=$(aws iam list-roles \
  --query 'Roles[?contains(RoleName,`NodeInstanceRole`)].RoleName' \
  --output text | tr '\t' '\n' | grep eksctl | head -1)

EFS_NODE_POLICY_ARN=$(aws iam create-policy \
  --policy-name EKS-EFS-Node-Policy \
  --policy-document file:///tmp/efs-node-policy.json \
  --query Policy.Arn --output text)

aws iam attach-role-policy \
  --role-name ${NODE_ROLE} \
  --policy-arn ${EFS_NODE_POLICY_ARN}

echo "节点 EFS 权限已添加"
```

**预期输出**：打印"节点 EFS 权限已添加"

### 7. 安装 EFS CSI Driver addon

```bash
aws eks create-addon \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name aws-efs-csi-driver \
  --pod-identity-associations "serviceAccount=efs-csi-controller-sa,roleArn=${EFS_ROLE_ARN}" \
  --region ${AWS_REGION}

until [ "$(aws eks describe-addon --cluster-name ${CLUSTER_NAME} --addon-name aws-efs-csi-driver \
  --region ${AWS_REGION} --query 'addon.status' --output text)" = "ACTIVE" ]; do
  echo "等待 EFS CSI addon ACTIVE..."
  sleep 15
done
echo "EFS CSI addon ACTIVE"
```

**预期输出**：`EFS CSI addon ACTIVE`

### 8. 创建 EFS 文件系统和安全组

```bash
VPC_ID=$(aws eks describe-cluster --name ${CLUSTER_NAME} \
  --query 'cluster.resourcesVpcConfig.vpcId' --output text)
CLUSTER_SG=$(aws eks describe-cluster --name ${CLUSTER_NAME} \
  --query 'cluster.resourcesVpcConfig.clusterSecurityGroupId' --output text)

EFS_SG_ID=$(aws ec2 create-security-group \
  --group-name efs-demo-sg \
  --description "EFS security group for EKS demo" \
  --vpc-id ${VPC_ID} \
  --query GroupId --output text)

aws ec2 authorize-security-group-ingress \
  --group-id ${EFS_SG_ID} \
  --protocol tcp --port 2049 \
  --source-group ${CLUSTER_SG}

EFS_ID=$(aws efs create-file-system \
  --performance-mode generalPurpose \
  --region ${AWS_REGION} \
  --query FileSystemId --output text)

echo "EFS ID: ${EFS_ID}"
```

**预期输出**：打印 EFS ID（格式 `fs-xxxxxxxx`）

### 9. 创建 EFS 挂载目标

```bash
PRIVATE_SUBNETS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=${VPC_ID}" \
           "Name=tag:kubernetes.io/role/internal-elb,Values=1" \
  --query 'Subnets[*].SubnetId' --output text)

for SUBNET in ${PRIVATE_SUBNETS}; do
  aws efs create-mount-target \
    --file-system-id ${EFS_ID} \
    --subnet-id ${SUBNET} \
    --security-groups ${EFS_SG_ID} \
    --region ${AWS_REGION}
  echo "挂载目标已创建：${SUBNET}"
done

sleep 30
echo "EFS 挂载目标已就绪"
```

**预期输出**：每个子网打印"挂载目标已创建"，最后打印"EFS 挂载目标已就绪"

### 10. 部署 EFS 共享挂载验证

```bash
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: ${EFS_ID}
  directoryPerms: "700"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 1Gi
EOF

kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: efs-writer
spec:
  containers:
  - name: writer
    image: public.ecr.aws/docker/library/busybox:latest
    command: ["sh", "-c", "echo 'written by pod1' > /data/shared.txt && sleep 3600"]
    volumeMounts:
    - name: efs-vol
      mountPath: /data
  volumes:
  - name: efs-vol
    persistentVolumeClaim:
      claimName: efs-claim
---
apiVersion: v1
kind: Pod
metadata:
  name: efs-reader
spec:
  containers:
  - name: reader
    image: public.ecr.aws/docker/library/busybox:latest
    command: ["sh", "-c", "sleep 10 && cat /data/shared.txt && sleep 3600"]
    volumeMounts:
    - name: efs-vol
      mountPath: /data
  volumes:
  - name: efs-vol
    persistentVolumeClaim:
      claimName: efs-claim
EOF

kubectl wait --for=condition=Ready pod/efs-writer pod/efs-reader --timeout=5m
sleep 15
kubectl logs efs-reader
```

**预期输出**：`written by pod1`（两个 Pod 共享同一 EFS 文件）

---

## 第三部分：S3 CSI Driver

### 11. 创建 S3 桶和 S3 Policy

```bash
S3_BUCKET="eks-csi-demo-${ACCOUNT_ID}"
aws s3api create-bucket --bucket ${S3_BUCKET} --region ${AWS_REGION}

S3_POLICY_ARN=$(aws iam create-policy \
  --policy-name EKS-S3-CSI-Policy \
  --policy-document "{
    \"Version\":\"2012-10-17\",
    \"Statement\":[{
      \"Effect\":\"Allow\",
      \"Action\":[\"s3:GetObject\",\"s3:PutObject\",\"s3:DeleteObject\",\"s3:ListBucket\"],
      \"Resource\":[\"arn:aws:s3:::${S3_BUCKET}\",\"arn:aws:s3:::${S3_BUCKET}/*\"]
    }]}" \
  --query Policy.Arn --output text)

NODE_ROLE=$(aws iam list-roles \
  --query 'Roles[?contains(RoleName,`NodeInstanceRole`)].RoleName' \
  --output text | tr '\t' '\n' | grep eksctl | head -1)
aws iam attach-role-policy \
  --role-name ${NODE_ROLE} \
  --policy-arn ${S3_POLICY_ARN}

echo "S3 桶和 Policy 已创建，Policy 已附加到节点 Role"
```

**预期输出**：打印确认信息

> ⚠️ S3 CSI Driver 使用节点实例角色访问 S3（不支持 Pod Identity），S3 Policy 必须直接附加到节点 IAM Role。

### 12. 安装 S3 CSI Driver 并验证挂载

```bash
aws eks create-addon \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name aws-mountpoint-s3-csi-driver \
  --region ${AWS_REGION}

until [ "$(aws eks describe-addon --cluster-name ${CLUSTER_NAME} \
  --addon-name aws-mountpoint-s3-csi-driver \
  --region ${AWS_REGION} --query 'addon.status' --output text)" = "ACTIVE" ]; do
  echo "等待 S3 CSI addon ACTIVE..."
  sleep 15
done
echo "S3 CSI addon ACTIVE"
```

**预期输出**：`S3 CSI addon ACTIVE`

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: s3-pv
spec:
  capacity:
    storage: 1200Gi
  accessModes:
    - ReadWriteMany
  mountOptions:
    - allow-delete
    - region us-east-1
  csi:
    driver: s3.csi.aws.com
    volumeHandle: s3-csi-driver-volume
    volumeAttributes:
      bucketName: ${S3_BUCKET}
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: s3-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 1200Gi
  volumeName: s3-pv
EOF

kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: s3-writer
spec:
  containers:
  - name: writer
    image: public.ecr.aws/docker/library/busybox:latest
    command: ["sh", "-c", "echo 'hello from EKS S3 CSI' > /s3/test.txt && echo 'Written!' && sleep 3600"]
    volumeMounts:
    - name: s3-vol
      mountPath: /s3
  volumes:
  - name: s3-vol
    persistentVolumeClaim:
      claimName: s3-claim
EOF

kubectl wait --for=condition=Ready pod/s3-writer --timeout=5m
sleep 10
kubectl logs s3-writer
```

**预期输出**：`Written!`

验证 S3 中有文件：

```bash
aws s3 ls s3://${S3_BUCKET}/
```

**预期输出**：显示 `test.txt` 文件

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 通过 EBS CSI Driver 动态创建 gp3 卷并挂载到 Pod 进行持久化写入
- [ ] 通过 EFS CSI Driver 实现多个 Pod 共享读写同一文件系统
- [ ] 通过 S3 Mountpoint CSI Driver 将 S3 桶挂载为 Pod 内文件系统

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws eks describe-addon --cluster-name demo --addon-name aws-ebs-csi-driver --region us-east-1 --query 'addon.status' --output text` | `ACTIVE` |
| 2 | `aws eks describe-addon --cluster-name demo --addon-name aws-efs-csi-driver --region us-east-1 --query 'addon.status' --output text` | `ACTIVE` |
| 3 | `aws eks describe-addon --cluster-name demo --addon-name aws-mountpoint-s3-csi-driver --region us-east-1 --query 'addon.status' --output text` | `ACTIVE` |
| 4 | `kubectl exec ebs-writer -- test -f /data/timestamps.txt && echo "exists"` | `exists` |
| 5 | `kubectl logs efs-reader 2>/dev/null \| head -1` | `written by pod1` |

---

## 实验总结

本实验通过三种 CSI 驱动演示了 EKS 中不同存储类型的集成方式：EBS 提供高性能块存储适合数据库场景，EFS 提供共享文件存储适合多 Pod 协作，S3 Mountpoint 提供海量对象存储的直接挂载。理解 Pod Identity 与节点实例角色两种权限模型的差异是正确配置 CSI 驱动的关键。

---

## 清理

```bash
kubectl delete pod ebs-writer efs-writer efs-reader s3-writer 2>/dev/null || true
kubectl delete pvc ebs-claim efs-claim s3-claim 2>/dev/null || true
kubectl delete pv s3-pv 2>/dev/null || true
kubectl delete storageclass ebs-sc efs-sc 2>/dev/null || true

aws eks delete-addon --cluster-name ${CLUSTER_NAME} --addon-name aws-mountpoint-s3-csi-driver --region ${AWS_REGION} 2>/dev/null || true
aws eks delete-addon --cluster-name ${CLUSTER_NAME} --addon-name aws-efs-csi-driver --region ${AWS_REGION} 2>/dev/null || true
aws eks delete-addon --cluster-name ${CLUSTER_NAME} --addon-name aws-ebs-csi-driver --region ${AWS_REGION} 2>/dev/null || true

S3_BUCKET="eks-csi-demo-${ACCOUNT_ID}"
aws s3 rm s3://${S3_BUCKET}/ --recursive 2>/dev/null || true
aws s3api delete-bucket --bucket ${S3_BUCKET} 2>/dev/null || true

EFS_ID=$(aws efs describe-file-systems --query 'FileSystems[0].FileSystemId' --output text)
for MT in $(aws efs describe-mount-targets --file-system-id ${EFS_ID} --query 'MountTargets[*].MountTargetId' --output text 2>/dev/null); do
  aws efs delete-mount-target --mount-target-id ${MT}
done
sleep 30
aws efs delete-file-system --file-system-id ${EFS_ID} 2>/dev/null || true

EFS_SG_ID=$(aws ec2 describe-security-groups --filters "Name=group-name,Values=efs-demo-sg" --query 'SecurityGroups[0].GroupId' --output text)
aws ec2 delete-security-group --group-id ${EFS_SG_ID} 2>/dev/null || true

NODE_ROLE=$(aws iam list-roles --query 'Roles[?contains(RoleName,`NodeInstanceRole`)].RoleName' --output text | tr '\t' '\n' | grep eksctl | head -1)
aws iam detach-role-policy --role-name ${NODE_ROLE} --policy-arn $(aws iam list-policies --query "Policies[?PolicyName=='EKS-S3-CSI-Policy'].Arn" --output text) 2>/dev/null || true
aws iam detach-role-policy --role-name ${NODE_ROLE} --policy-arn $(aws iam list-policies --query "Policies[?PolicyName=='EKS-EFS-Node-Policy'].Arn" --output text) 2>/dev/null || true
aws iam delete-policy --policy-arn $(aws iam list-policies --query "Policies[?PolicyName=='EKS-S3-CSI-Policy'].Arn" --output text) 2>/dev/null || true
aws iam delete-policy --policy-arn $(aws iam list-policies --query "Policies[?PolicyName=='EKS-EFS-Node-Policy'].Arn" --output text) 2>/dev/null || true

aws iam detach-role-policy --role-name AmazonEKS_EBS_CSI_DriverRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy 2>/dev/null || true
aws iam detach-role-policy --role-name AmazonEKS_EFS_CSI_DriverRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEFSCSIDriverPolicy 2>/dev/null || true
aws iam delete-role --role-name AmazonEKS_EBS_CSI_DriverRole 2>/dev/null || true
aws iam delete-role --role-name AmazonEKS_EFS_CSI_DriverRole 2>/dev/null || true

echo "清理完成"
```
