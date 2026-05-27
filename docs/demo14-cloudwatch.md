# Demo14 — CloudWatch Observability 原生可观测性

## 实验简介

AWS CloudWatch Observability 是 EKS 的原生可观测性方案，通过 addon 一键部署即可实现控制面日志、节点指标和应用日志的统一采集。相比自建 Prometheus 栈，CloudWatch 方案无需管理存储和高可用，适合追求运维简洁性的团队。本实验将启用 Control Plane 日志并安装 CloudWatch Observability addon。

**实验目标：**
- 掌握 EKS Control Plane 日志的启用与验证方法
- 理解 CloudWatch Observability addon 的 DaemonSet 架构
- 能够独立完成从日志生成到 CloudWatch Logs 查询的完整链路验证

**实验流程：**
1. 启用 EKS Control Plane 五类日志
2. 安装 CloudWatch Observability addon 并验证 DaemonSet
3. 部署测试 Pod 生成应用日志
4. 在 CloudWatch Logs 中查询控制面和应用日志

**预计 AI 执行时长：** 8-10 分钟

---

## 前提条件

- **工具**：AWS CLI v2、eksctl、kubectl v1.35
- **权限**：AdministratorAccess（含 CloudWatch Logs 写权限）
- **前提**：Demo01 已完成
- **费用提示**：CloudWatch Logs 按量计费，实验结束须立即删除日志组
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export CLUSTER_NAME=demo
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

---

## 步骤

### 1. 启用 EKS Control Plane 日志

将配置写入文件再传参（避免 shell 转义问题）：

```bash
# 检查日志是否已启用，已启用则跳过
LOGGING_ENABLED=$(aws eks describe-cluster --name ${CLUSTER_NAME} --region ${AWS_REGION} \
  --query 'cluster.logging.clusterLogging[?enabled==`true`].types[0]' --output text 2>/dev/null)

if [ -z "${LOGGING_ENABLED}" ] || [ "${LOGGING_ENABLED}" = "None" ]; then
  cat > /tmp/logging.json << 'EOF'
{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}
EOF
  aws eks update-cluster-config \
    --name ${CLUSTER_NAME} \
    --logging file:///tmp/logging.json \
    --region ${AWS_REGION}
  echo "Control plane 日志启用中，等待集群更新..."
else
  echo "Control plane 日志已启用，跳过"
fi
```

**预期输出**：`Control plane 日志启用中...` 或 `已启用，跳过`

> ⚠️ 必须用 `file://` 方式传参，直接内联 JSON 字符串容易因 shell 转义导致参数解析失败。

### 2. 等待集群更新完成

```bash
# 仅在上一步触发了更新时需要等待
CLUSTER_STATUS=$(aws eks describe-cluster --name ${CLUSTER_NAME} --region ${AWS_REGION} --query 'cluster.status' --output text)
if [ "${CLUSTER_STATUS}" != "ACTIVE" ]; then
  until [ "$(aws eks describe-cluster --name ${CLUSTER_NAME} \
    --region ${AWS_REGION} \
    --query 'cluster.status' --output text)" = "ACTIVE" ]; do
    echo "等待集群更新..."
    sleep 15
  done
fi
echo "集群状态：ACTIVE"
```

**预期输出**：`集群状态：ACTIVE`

### 3. 验证日志启用状态

```bash
aws eks describe-cluster --name ${CLUSTER_NAME} \
  --region ${AWS_REGION} \
  --query 'cluster.logging.clusterLogging' \
  --output json
```

**预期输出**：JSON 数组，显示各日志类型 `"enabled": true`

> ⚠️ 必须用 `--output json`，`--output table` 渲染 `enabled` 字段不准确，会显示 `False`。

### 4. 创建 CloudWatch Agent IAM 角色与 Pod Identity 关联

addon 需要 IAM 权限才能向 CloudWatch Logs 写入数据，使用 Pod Identity 方式授权：

```bash
cat > /tmp/cw-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "pods.eks.amazonaws.com"
      },
      "Action": ["sts:AssumeRole", "sts:TagSession"]
    }
  ]
}
EOF

aws iam create-role \
  --role-name CloudWatchAgentRole-${CLUSTER_NAME} \
  --assume-role-policy-document file:///tmp/cw-trust-policy.json \
  --region ${AWS_REGION}

aws iam attach-role-policy \
  --role-name CloudWatchAgentRole-${CLUSTER_NAME} \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

aws iam attach-role-policy \
  --role-name CloudWatchAgentRole-${CLUSTER_NAME} \
  --policy-arn arn:aws:iam::aws:policy/AWSXrayWriteOnlyAccess

aws eks create-pod-identity-association \
  --cluster-name ${CLUSTER_NAME} \
  --namespace amazon-cloudwatch \
  --service-account cloudwatch-agent \
  --role-arn arn:aws:iam::${ACCOUNT_ID}:role/CloudWatchAgentRole-${CLUSTER_NAME} \
  --region ${AWS_REGION}
```

**预期输出**：角色创建成功，Pod Identity 关联创建成功

> ⚠️ 必须在安装 addon 之前或之后立即创建 Pod Identity 关联，否则 fluent-bit 会因 AccessDeniedException 无法推送日志。如果 addon 已安装但日志未出现，创建关联后需重启 DaemonSet：`kubectl rollout restart daemonset cloudwatch-agent fluent-bit -n amazon-cloudwatch`

### 5. 安装 CloudWatch Observability addon

```bash
eksctl create addon \
  --cluster ${CLUSTER_NAME} \
  --name amazon-cloudwatch-observability \
  --force

until [ "$(aws eks describe-addon --cluster-name ${CLUSTER_NAME} \
  --addon-name amazon-cloudwatch-observability \
  --region ${AWS_REGION} \
  --query 'addon.status' --output text)" = "ACTIVE" ]; do
  echo "等待 CloudWatch addon ACTIVE..."
  sleep 15
done
echo "CloudWatch Observability addon ACTIVE"
```

**预期输出**：`CloudWatch Observability addon ACTIVE`

### 6. 验证 CloudWatch Agent DaemonSet

```bash
kubectl get daemonset -n amazon-cloudwatch
kubectl get pods -n amazon-cloudwatch
```

**预期输出**：每个节点上均有 CloudWatch Agent Pod 在运行（DESIRED = READY = 节点数）

### 7. 部署测试 Pod 生成应用日志

```bash
kubectl create deployment log-generator \
  --image=public.ecr.aws/docker/library/busybox:latest \
  -- sh -c 'i=0; while true; do echo "[$(date)] Log line $i from demo14 test pod"; i=$((i+1)); sleep 1; done'

kubectl rollout status deployment/log-generator --timeout=2m
echo "日志生成器已启动，等待 60 秒让日志积累..."
sleep 60
```

**预期输出**：Deployment rollout 成功

### 8. 查看控制面日志组

```bash
aws logs describe-log-groups \
  --log-group-name-prefix "/aws/eks/${CLUSTER_NAME}" \
  --region ${AWS_REGION} \
  --query 'logGroups[*].logGroupName' \
  --output text
```

**预期输出**：显示 `/aws/eks/${CLUSTER_NAME}/cluster` 日志组

> ⚠️ audit log stream 需等待约 5 分钟才开始出现，立即查询可能为空，属正常现象。

### 9. 查看应用日志

```bash
aws logs describe-log-groups \
  --log-group-name-prefix "/aws/containerinsights/${CLUSTER_NAME}" \
  --region ${AWS_REGION} \
  --query 'logGroups[*].logGroupName' \
  --output text
```

**预期输出**：显示 Container Insights 日志组（含 `/application`、`/dataplane` 等）

```bash
LOG_STREAM=$(aws logs describe-log-streams \
  --log-group-name "/aws/containerinsights/${CLUSTER_NAME}/application" \
  --order-by LastEventTime --descending --limit 1 \
  --query 'logStreams[0].logStreamName' --output text \
  --region ${AWS_REGION} 2>/dev/null)

if [ -n "${LOG_STREAM}" ] && [ "${LOG_STREAM}" != "None" ]; then
  aws logs get-log-events \
    --log-group-name "/aws/containerinsights/${CLUSTER_NAME}/application" \
    --log-stream-name "${LOG_STREAM}" \
    --limit 5 \
    --query 'events[*].message' \
    --output text \
    --region ${AWS_REGION}
else
  echo "日志尚未推送，等待约 2-3 分钟后重试"
fi
```

**预期输出**：显示包含 `Log line` 的应用日志条目

---

## 验收标准

完成本实验后，你应当能够：
- [ ] CloudWatch Observability addon 状态为 ACTIVE，DaemonSet Pod 覆盖所有节点
- [ ] Control Plane 日志组 `/aws/eks/demo/cluster` 已创建
- [ ] Container Insights 应用日志组中能查询到测试 Pod 的日志条目

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws eks describe-addon --cluster-name demo --addon-name amazon-cloudwatch-observability --region us-east-1 --query 'addon.status' --output text` | `ACTIVE` |
| 2 | `aws logs describe-log-groups --log-group-name-prefix "/aws/eks/demo" --region us-east-1 --query 'length(logGroups)' --output text` | `1` |
| 3 | `kubectl get daemonset cloudwatch-agent -n amazon-cloudwatch -o jsonpath='{.status.numberReady}' 2>/dev/null \|\| kubectl get daemonset -n amazon-cloudwatch --no-headers \| head -1 \| awk '{print $4}'` | 等于节点数（`kubectl get nodes --no-headers \| wc -l`） |

---

## 实验总结

本实验完成了「CloudWatch Observability 原生可观测性」的全部操作，从资源创建到功能验证形成了完整闭环。通过动手实践，你已掌握了相关组件的核心配置方法和最佳实践。Demo15 将学习管理不同计算节点类型与 Fargate。

---

## 清理

```bash
kubectl delete deployment log-generator 2>/dev/null || true

eksctl delete addon --cluster ${CLUSTER_NAME} --name amazon-cloudwatch-observability 2>/dev/null || true
kubectl delete namespace amazon-cloudwatch 2>/dev/null || true

cat > /tmp/logging-disable.json << 'EOF'
{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":false}]}
EOF
aws eks update-cluster-config \
  --name ${CLUSTER_NAME} \
  --logging file:///tmp/logging-disable.json \
  --region ${AWS_REGION}

until [ "$(aws eks describe-cluster --name ${CLUSTER_NAME} \
  --region ${AWS_REGION} --query 'cluster.status' --output text)" = "ACTIVE" ]; do sleep 10; done

for LG in $(aws logs describe-log-groups \
  --log-group-name-prefix "/aws/eks/${CLUSTER_NAME}" \
  --region ${AWS_REGION} \
  --query 'logGroups[*].logGroupName' --output text); do
  aws logs delete-log-group --log-group-name "${LG}" --region ${AWS_REGION}
  echo "已删除日志组: ${LG}"
done

for LG in $(aws logs describe-log-groups \
  --log-group-name-prefix "/aws/containerinsights/${CLUSTER_NAME}" \
  --region ${AWS_REGION} \
  --query 'logGroups[*].logGroupName' --output text); do
  aws logs delete-log-group --log-group-name "${LG}" --region ${AWS_REGION}
  echo "已删除日志组: ${LG}"
done

aws eks delete-pod-identity-association \
  --cluster-name ${CLUSTER_NAME} \
  --association-id $(aws eks list-pod-identity-associations \
    --cluster-name ${CLUSTER_NAME} --namespace amazon-cloudwatch \
    --service-account cloudwatch-agent --region ${AWS_REGION} \
    --query 'associations[0].associationId' --output text) \
  --region ${AWS_REGION} 2>/dev/null || true

aws iam detach-role-policy --role-name CloudWatchAgentRole-${CLUSTER_NAME} \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy 2>/dev/null || true
aws iam detach-role-policy --role-name CloudWatchAgentRole-${CLUSTER_NAME} \
  --policy-arn arn:aws:iam::aws:policy/AWSXrayWriteOnlyAccess 2>/dev/null || true
aws iam delete-role --role-name CloudWatchAgentRole-${CLUSTER_NAME} 2>/dev/null || true

echo "清理完成"
```
