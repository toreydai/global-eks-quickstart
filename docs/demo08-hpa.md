# Demo08 — 使用 HPA 进行自动伸缩

## 实验简介

Horizontal Pod Autoscaler (HPA) 是 Kubernetes 内置的 Pod 水平自动伸缩机制，根据 CPU/内存等指标动态调整副本数。本实验通过部署 Metrics Server、创建 HPA 策略并模拟负载，完整演示从扩容到缩容的自动伸缩生命周期。

**实验目标：**
- 掌握 Metrics Server 的安装与验证方法
- 理解 HPA 基于 CPU 利用率的扩缩容决策逻辑
- 能够独立完成 HPA 配置、负载测试与缩容观察

**实验流程：**
1. 安装 Metrics Server EKS Addon 并验证指标采集
2. 部署 php-apache 应用并创建 HPA 策略
3. 启动负载生成器观察自动扩容
4. 停止负载观察自动缩容

**预计 AI 执行时长：** 8-10 分钟

---

## 前提条件

- **工具**：kubectl、eksctl、aws CLI
- **权限**：EKS 操作权限（含 Addon 安装权限）
- **前提**：Demo01 集群已创建（3 个节点）
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

### 1. 安装 Metrics Server（EKS 托管 Addon）

```bash
aws eks create-addon \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name metrics-server \
  --region ${AWS_REGION}

# 轮询直到 ACTIVE
while true; do
  STATUS=$(aws eks describe-addon \
    --cluster-name ${CLUSTER_NAME} \
    --addon-name metrics-server \
    --region ${AWS_REGION} \
    --query 'addon.status' --output text)
  echo "Metrics Server 状态：${STATUS}"
  [[ "${STATUS}" == "ACTIVE" ]] && break
  sleep 15
done
```

**预期输出**：`Metrics Server 状态：ACTIVE`

### 2. 验证 Metrics Server 可用

```bash
# 等待 30s 让 metrics 采集就绪
sleep 30
kubectl top nodes
```

**预期输出**：表格输出，每行显示节点 CPU/Memory 使用量（非 `<unknown>`）

### 3. 创建 PHP 应用 ConfigMap

```bash
kubectl create configmap php-apache-index --from-literal=index.php='<?php
  $x = 0.0001;
  for ($i = 0; $i <= 1000000; $i++) { $x += sqrt($x); }
  echo "OK!";
?>'
```

**预期输出**：`configmap/php-apache-index created`

### 4. 部署 php-apache 应用（含 ConfigMap 挂载）

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: public.ecr.aws/docker/library/php:apache
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
        volumeMounts:
        - name: php-config
          mountPath: /var/www/html/index.php
          subPath: index.php
      volumes:
      - name: php-config
        configMap:
          name: php-apache-index
EOF
```

**预期输出**：`deployment.apps/php-apache created`

> ⚠️ PHP 代码必须通过 ConfigMap volume 挂载到 `/var/www/html/index.php`，仅用 `kubectl set env` 或 args 方式无法生效。

### 5. 暴露 Service 并等待 Deployment 就绪

```bash
kubectl expose deployment php-apache --port 80
kubectl rollout status deployment/php-apache --timeout=5m
```

**预期输出**：`deployment "php-apache" successfully rolled out`

### 6. 创建 HPA（CPU 阈值 50%，最多 10 个 Pod）

```bash
kubectl autoscale deployment php-apache \
  --cpu-percent=50 \
  --min=1 \
  --max=10 \
```

**预期输出**：`horizontalpodautoscaler.autoscaling/php-apache autoscaled`

> ⚠️ 使用 `--cpu-percent=50`（或新版 `--cpu=50%`）。缩容稳定窗口默认为 5 分钟（由 kube-controller-manager 控制），无法通过 kubectl autoscale 命令行参数修改。

```bash
# 等待 HPA 获取到当前 CPU 指标
sleep 30
kubectl get hpa php-apache
```

**预期输出**：TARGETS 列显示 `X%/50%`（数值非 `<unknown>`），MINPODS=1，MAXPODS=10

### 7. 启动负载生成器（新终端或后台）

```bash
# 在当前终端后台运行负载生成器（也可在新终端运行）
kubectl run load-generator \
  --image=public.ecr.aws/amazonlinux/amazonlinux:latest \
  --restart=Never \
  -- /bin/bash -c "while true; do curl -s http://php-apache > /dev/null; done" &
echo "负载生成器已启动"
```

**预期输出**：`pod/load-generator created`，`负载生成器已启动`

### 8. 观察 HPA 扩容

```bash
# 轮询直到 REPLICAS > 1（约 2-3 分钟）
echo "等待 HPA 触发扩容..."
for i in $(seq 1 12); do
  REPLICAS=$(kubectl get hpa php-apache -o jsonpath='{.status.currentReplicas}' 2>/dev/null)
  echo "当前副本数：${REPLICAS}"
  [[ "${REPLICAS}" -gt 1 ]] 2>/dev/null && echo "✅ 扩容已触发" && break
  sleep 15
done
kubectl get hpa php-apache
```

**预期输出**：TARGETS 超过 `50%`，REPLICAS 增加到最多 10；pods 数量跟随增加

### 9. 停止负载生成器，验证缩容触发

```bash
kubectl delete pod load-generator --wait=false 2>/dev/null || kill %1 2>/dev/null || true
echo "负载生成器已停止"

# 验证 HPA 已感知负载下降（无需等待完全缩容）
sleep 30
kubectl get hpa php-apache
echo "缩容将在约 5 分钟后自动完成（受 downscale-stabilization 窗口控制）"
```

**预期输出**：TARGETS 开始降低（可能仍 >50%），说明 HPA 已感知负载变化。完全缩回 1 需约 5 分钟。

---

## 验收标准

完成本实验后，你应当能够：
- [ ] Metrics Server Addon 状态为 ACTIVE，kubectl top nodes 正常输出指标
- [ ] HPA 创建成功且 TARGETS 列显示实际 CPU 百分比（非 unknown）
- [ ] 负载期间 HPA 自动将副本数从 1 扩容至多个 Pod
- [ ] 负载停止后副本数自动缩回 1

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `aws eks describe-addon --cluster-name demo --addon-name metrics-server --region us-east-1 --query 'addon.status' --output text` | `ACTIVE` |
| 2 | `kubectl get hpa php-apache -o jsonpath='{.spec.minReplicas}'` | `1` |
| 3 | `kubectl get hpa php-apache -o jsonpath='{.spec.maxReplicas}'` | `10` |
| 4 | `kubectl get hpa php-apache -o jsonpath='{.spec.metrics[0].resource.target.averageUtilization}'` | `50` |
| 5 | `kubectl get deployment php-apache -o jsonpath='{.status.readyReplicas}'` | `1`（初始状态，负载停止后缩回） |

---

## 实验总结

本实验完成了 HPA 自动伸缩的端到端验证：安装 Metrics Server 提供指标数据源，配置 HPA 策略定义扩缩容阈值，通过负载测试观察了 Pod 从 1 副本扩容到多副本再缩回的完整生命周期。HPA 解决了 Pod 级别的弹性伸缩，但当节点资源不足时需要节点级别的自动伸缩，下一个实验将介绍 Karpenter 实现节点自动扩缩容。

---

## 清理

```bash
# 停止负载生成器
kubectl delete pod load-generator 2>/dev/null || true

# 删除 HPA、Deployment、Service
kubectl delete hpa php-apache 2>/dev/null || true
kubectl delete deployment php-apache 2>/dev/null || true
kubectl delete svc php-apache 2>/dev/null || true
kubectl delete configmap php-apache-index 2>/dev/null || true

# 删除 Metrics Server addon（可选，其他 Demo 可能需要）
# aws eks delete-addon --cluster-name demo --addon-name metrics-server --region us-east-1

echo "清理完成"
```
