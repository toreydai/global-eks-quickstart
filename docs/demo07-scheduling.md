# Demo07 — 调度策略与资源治理

## 实验简介

Kubernetes 调度器负责将 Pod 分配到合适的节点，而资源治理机制确保集群资源不被滥用。本实验通过 ResourceQuota、LimitRange、nodeSelector、Affinity、Taints/Tolerations 和 PriorityClass 等核心机制，全面掌握 Pod 调度与资源管控能力。

**实验目标：**
- 掌握 ResourceQuota 和 LimitRange 的资源治理配置
- 理解 nodeSelector、Affinity、TopologySpreadConstraints 和 Taints/Tolerations 的调度策略差异
- 能够独立完成 PriorityClass 抢占调度的配置与验证

**实验流程：**
1. 配置 LimitRange 和 ResourceQuota 实现资源治理
2. 使用 nodeSelector 和 Anti-affinity 控制 Pod 调度位置
3. 通过 TopologySpreadConstraints 实现跨 zone 均匀分布
4. 配置 Taints/Tolerations 实现专用节点隔离
5. 创建 PriorityClass 验证高优先级抢占行为

**预计 AI 执行时长：** 10-12 分钟

---

## 前提条件

- **工具**：kubectl
- **权限**：EKS 操作权限
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

## 第一部分：资源治理

### 1. 创建专用 namespace

```bash
kubectl create namespace scheduling-demo
```

**预期输出**：`namespace/scheduling-demo created`

### 2. 创建 LimitRange 和 ResourceQuota

```bash
kubectl apply -n scheduling-demo -f - <<'EOF'
apiVersion: v1
kind: LimitRange
metadata:
  name: demo-limits
spec:
  limits:
  - type: Container
    default:
      cpu: "200m"
      memory: "128Mi"
    defaultRequest:
      cpu: "100m"
      memory: "64Mi"
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: demo-quota
spec:
  hard:
    pods: "30"
    requests.cpu: "4"
    requests.memory: "4Gi"
EOF
```

**预期输出**：`limitrange/demo-limits created`，`resourcequota/demo-quota created`

> ⚠️ ResourceQuota pods=30 是因为多步演示累计 Pod 数量多。每个步骤：制造 → 验证 → 理解。

### 3. 验证超配额被拒绝

```bash
# 创建占用大量 CPU 的 Deployment（需同时设置 limits 以绕过 LimitRange 默认限制）
kubectl apply -n scheduling-demo -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quota-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: quota-test
  template:
    metadata:
      labels:
        app: quota-test
    spec:
      containers:
      - name: nginx
        image: public.ecr.aws/docker/library/nginx:1.27-alpine
        resources:
          requests:
            cpu: "1500m"
            memory: "64Mi"
          limits:
            cpu: "1500m"
            memory: "128Mi"
EOF

# 扩容到 4 副本（4×1500m=6>4 核，超出 quota）
kubectl scale deployment quota-test -n scheduling-demo --replicas=4
sleep 5

# 应看到 FailedCreate: exceeded quota
kubectl get events -n scheduling-demo --sort-by='.lastTimestamp' | tail -5
```

**预期输出**：Events 中显示 `exceeded quota` 或 `FailedCreate` 事件

```bash
# 验证实际只有 1 个 Pod 运行（其余因 quota 被拒绝）
kubectl get deployment quota-test -n scheduling-demo
```

**预期输出**：READY 列为 `2/4`（仅 2 个副本启动，因 2×1500m=3000m < 4 核 quota，第 3 个超额被拒绝）

```bash
# 查看当前 quota 用量
kubectl get resourcequota demo-quota -n scheduling-demo --output=yaml | grep -A10 'used:'
```

**预期输出**：显示 `used:` 下的 CPU/memory/pods 当前用量

---

## 第二部分：调度策略

### 4. nodeSelector — 定向调度

```bash
# 给第一个节点打标签
TARGET_NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl label node ${TARGET_NODE} demo-role=target
echo "目标节点：${TARGET_NODE}"
```

**预期输出**：`node/<node-name> labeled`，并输出节点名称

```bash
kubectl apply -n scheduling-demo -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: nodeselector-pod
spec:
  nodeSelector:
    demo-role: target
  containers:
  - name: nginx
    image: public.ecr.aws/docker/library/nginx:1.27-alpine
EOF

kubectl wait pod nodeselector-pod -n scheduling-demo --for=condition=Ready --timeout=60s
kubectl get pod nodeselector-pod -n scheduling-demo -o wide
```

**预期输出**：Pod 状态 `Running`，NODE 列显示目标节点名称

```bash
# 验证调度事件（Scheduler 选择了目标节点）
kubectl get events -n scheduling-demo --field-selector reason=Scheduled | grep nodeselector
```

**预期输出**：事件显示 Pod 被调度到带 `demo-role=target` 标签的节点

### 5. Pod Anti-affinity — 每节点最多 1 Pod

```bash
kubectl apply -n scheduling-demo -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: anti-affinity-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: anti-demo
  template:
    metadata:
      labels:
        app: anti-demo
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: anti-demo
            topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: public.ecr.aws/docker/library/nginx:1.27-alpine
EOF

kubectl rollout status deployment/anti-affinity-demo -n scheduling-demo --timeout=3m
kubectl get pods -n scheduling-demo -l app=anti-demo -o wide
```

**预期输出**：3 个 Pod 分布在 3 个不同节点（NODE 列无重复）

### 6. TopologySpreadConstraints — 跨 zone 均匀分散

```bash
kubectl apply -n scheduling-demo -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: topology-demo
spec:
  replicas: 4
  selector:
    matchLabels:
      app: topo-demo
  template:
    metadata:
      labels:
        app: topo-demo
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: topo-demo
      containers:
      - name: nginx
        image: public.ecr.aws/docker/library/nginx:1.27-alpine
EOF

kubectl rollout status deployment/topology-demo -n scheduling-demo --timeout=3m

# 查看各 zone 分布
kubectl get pods -n scheduling-demo -l app=topo-demo -o wide
kubectl get pods -n scheduling-demo -l app=topo-demo \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort | uniq -c
```

**预期输出**：4 个 Pod 全部 Running；各节点 Pod 数量较均匀分布（`ScheduleAnyway` 不强制均匀）

> ⚠️ 使用 `ScheduleAnyway` 避免有 taint 节点时 Pod Pending；`DoNotSchedule` 在节点较少时可能导致 Pending。

### 7. Taints / Tolerations — 专用节点

```bash
TAINT_NODE=$(kubectl get nodes -o jsonpath='{.items[1].metadata.name}')
kubectl taint node ${TAINT_NODE} dedicated=gpu:NoSchedule
echo "已对节点 ${TAINT_NODE} 添加 taint"
```

**预期输出**：`node/<node-name> tainted`

```bash
# 无 toleration Pod（不应调度到 taint 节点）
kubectl apply -n scheduling-demo -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: no-toleration
spec:
  containers:
  - name: nginx
    image: public.ecr.aws/docker/library/nginx:1.27-alpine
EOF

# 有 toleration Pod（可调度到 taint 节点）
TAINT_NODE_NAME=$(kubectl get nodes -o jsonpath='{.items[1].metadata.name}')
kubectl apply -n scheduling-demo -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
spec:
  tolerations:
  - key: dedicated
    value: gpu
    effect: NoSchedule
  nodeSelector:
    kubernetes.io/hostname: ${TAINT_NODE_NAME}
  containers:
  - name: nginx
    image: public.ecr.aws/docker/library/nginx:1.27-alpine
EOF

sleep 10
kubectl get pods -n scheduling-demo no-toleration with-toleration -o wide
```

**预期输出**：`no-toleration` Pod NODE 列不显示 taint 节点；`with-toleration` Pod 运行在 taint 节点上

---

## 第三部分：PriorityClass 与抢占

### 8. 创建 PriorityClass

```bash
kubectl apply -f - <<'EOF'
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: false
description: "低优先级任务"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 10000
globalDefault: false
description: "高优先级关键任务"
EOF
```

**预期输出**：`priorityclass.scheduling.k8s.io/low-priority created`，`priorityclass.scheduling.k8s.io/high-priority created`

### 9. 用低优先级 Pod 占满所有节点

```bash
# 独立 namespace 避开 ResourceQuota 干扰
kubectl create namespace preemption-demo

# 每个节点放一个大 CPU request 的低优先级 Pod（用 nodeName 确保分布）
NODES=($(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'))
for i in "${!NODES[@]}"; do
  kubectl apply -n preemption-demo -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: low-pri-${i}
  labels:
    tier: low
spec:
  priorityClassName: low-priority
  nodeName: ${NODES[$i]}
  containers:
  - name: nginx
    image: public.ecr.aws/docker/library/nginx:1.27-alpine
    resources:
      requests:
        cpu: "1500m"
        memory: "256Mi"
EOF
done

kubectl wait --for=condition=Ready -n preemption-demo -l tier=low pod --timeout=90s
echo "低优先级 Pod 已占满所有节点："
kubectl get pods -n preemption-demo -o wide
```

**预期输出**：所有 `low-pri-*` Pod 状态为 `Running`

### 10. 部署高优先级 Pod 触发抢占

```bash
# 高优先级 Pod：不指定 nodeName，交给 scheduler 触发抢占
kubectl apply -n preemption-demo -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: high-pri-pod
spec:
  priorityClassName: high-priority
  containers:
  - name: nginx
    image: public.ecr.aws/docker/library/nginx:1.27-alpine
    resources:
      requests:
        cpu: "1500m"
        memory: "256Mi"
EOF

echo "等待 scheduler 触发抢占..."
# 轮询直到 high-pri-pod Running（通常 10-20s）
for i in $(seq 1 6); do
  STATUS=$(kubectl get pod high-pri-pod -n preemption-demo -o jsonpath='{.status.phase}' 2>/dev/null)
  [[ "${STATUS}" == "Running" ]] && break
  sleep 5
done

echo "=== 抢占结果 ==="
kubectl get pods -n preemption-demo -o wide

echo "=== 抢占事件 ==="
kubectl get events -n preemption-demo --sort-by='.lastTimestamp' \
  | grep -E "Preempt|Evict|Schedul|kill" | head -10
```

**预期输出**：`high-pri-pod` 状态为 `Running`；至少一个 `low-pri-*` Pod 消失（被抢占/Terminating/Evicted）

---

## 验收标准

完成本实验后，你应当能够：
- [ ] ResourceQuota 成功限制超额 Pod 创建，Events 中出现 exceeded quota 事件
- [ ] nodeSelector Pod 被调度到指定标签节点
- [ ] Anti-affinity Deployment 的 3 个 Pod 分布在 3 个不同节点
- [ ] 高优先级 Pod 成功抢占低优先级 Pod 并进入 Running 状态

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl get resourcequota demo-quota -n scheduling-demo -o jsonpath='{.spec.hard.pods}'` | `30` |
| 2 | `kubectl get pod nodeselector-pod -n scheduling-demo -o jsonpath='{.spec.nodeName}'` | 节点名称与 `kubectl get nodes -o jsonpath='{.items[0].metadata.name}'` 输出一致 |
| 3 | `kubectl get pods -n scheduling-demo -l app=anti-demo -o jsonpath='{.items[*].spec.nodeName}' \| tr ' ' '\n' \| sort \| uniq \| wc -l \| tr -d ' '` | `3` |
| 4 | `kubectl get pod high-pri-pod -n preemption-demo -o jsonpath='{.status.phase}'` | `Running` |
| 5 | `kubectl get priorityclass high-priority -o jsonpath='{.value}'` | `10000` |

---

## 实验总结

本实验构建了完整的调度策略与资源治理体系：通过 ResourceQuota/LimitRange 防止资源滥用，通过 nodeSelector、Affinity 和 TopologySpreadConstraints 精确控制 Pod 分布，通过 Taints/Tolerations 实现节点隔离，通过 PriorityClass 实现关键负载的抢占保障。这些调度机制是生产环境多租户集群治理的基础，下一个实验将在此基础上引入 HPA，实现基于负载的 Pod 水平自动伸缩。

---

## 清理

```bash
# 删除抢占演示 namespace
kubectl delete namespace preemption-demo 2>/dev/null || true

# 删除调度演示 namespace（含所有 Pod 和 Deployment）
kubectl delete namespace scheduling-demo 2>/dev/null || true

# 移除节点标签
TARGET_NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl label node ${TARGET_NODE} demo-role- 2>/dev/null || true

# 移除节点 taint
TAINT_NODE=$(kubectl get nodes -o jsonpath='{.items[1].metadata.name}')
kubectl taint node ${TAINT_NODE} dedicated- 2>/dev/null || true

# 删除 PriorityClass（cluster-scoped）
kubectl delete priorityclass low-priority high-priority 2>/dev/null || true

echo "清理完成"
```
