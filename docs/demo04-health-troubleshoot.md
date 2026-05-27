# Demo04 — 健康检查与故障排查

## 实验简介

本实验将完成「健康检查与故障排查」的全部操作，涵盖资源创建、配置、验证的完整流程。

**实验目标：**
- 掌握 健康检查与故障排查 的核心操作流程
- 理解相关组件的工作原理和配置要点
- 能够独立完成从部署到验证的完整闭环

**实验流程：**
1. 创建专用 namespace
2. 部署带完整 Probe 的应用
3. 演示 liveness probe 失败导致容器重启
4. 演示 readiness probe 失败导致 Pod 从 Service 摘除
5. 制造并排查 ImagePullBackOff
6. 制造并排查 CrashLoopBackOff
7. 制造并排查 OOMKilled
8. 制造并排查 Pending（资源不足）
9. 制造并排查 Service selector 错误（Endpoints 为空）
10. 演示 kubectl debug 进入无 shell 容器排障

**预计 AI 执行时长：** 5-8 分钟


## 前提条件

- **工具**：kubectl、aws CLI
- **权限**：EKS 操作权限
- **前提**：Demo01 集群已创建
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

### 1. 创建专用 namespace

```bash
kubectl create namespace troubleshoot-demo
```

**预期输出**：`namespace/troubleshoot-demo created`

### 2. 部署带完整 Probe 的应用

```bash
kubectl apply -n troubleshoot-demo -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: probe-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: probe-demo
  template:
    metadata:
      labels:
        app: probe-demo
    spec:
      containers:
      - name: nginx
        image: public.ecr.aws/nginx/nginx:latest
        ports:
        - containerPort: 80
        startupProbe:
          httpGet:
            path: /
            port: 80
          failureThreshold: 30
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 10
          failureThreshold: 3
EOF

kubectl rollout status deployment/probe-demo -n troubleshoot-demo --timeout=2m
```

**预期输出**：`deployment "probe-demo" successfully rolled out`

### 3. 演示 liveness probe 失败导致容器重启

```bash
# 部署 liveness probe 故意失败的 Pod
kubectl apply -n troubleshoot-demo -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: liveness-fail
spec:
  containers:
  - name: app
    image: public.ecr.aws/docker/library/busybox:latest
    command: ["/bin/sh", "-c", "sleep 10; exit 1"]
    livenessProbe:
      exec:
        command: ["cat", "/tmp/healthy"]
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 2
EOF

# 等待约 30s 观察重启
sleep 35
kubectl get pod liveness-fail -n troubleshoot-demo
```

**预期输出**：`RESTARTS` 列数值 >= 1，状态可能为 `CrashLoopBackOff`

```bash
# 查看重启事件
kubectl describe pod liveness-fail -n troubleshoot-demo | grep -A5 "Events:"
```

**预期输出**：Events 中显示 `Liveness probe failed` 和 `Killing` 事件

### 4. 演示 readiness probe 失败导致 Pod 从 Service 摘除

```bash
kubectl apply -n troubleshoot-demo -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: readiness-fail
  labels:
    app: readiness-demo
spec:
  containers:
  - name: app
    image: public.ecr.aws/nginx/nginx:latest
    readinessProbe:
      httpGet:
        path: /nonexistent
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: readiness-demo-svc
spec:
  selector:
    app: readiness-demo
  ports:
  - port: 80
EOF

sleep 20
kubectl get pod readiness-fail -n troubleshoot-demo
kubectl get endpoints readiness-demo-svc -n troubleshoot-demo
```

**预期输出**：Pod 状态 `0/1 Running`；Endpoints 显示 `<none>` 或无地址（表示 Pod 已从 Service 摘除）

### 5. 制造并排查 ImagePullBackOff

```bash
# 制造：镜像不存在
kubectl run imgpull-fail -n troubleshoot-demo \
  --image=public.ecr.aws/nonexistent/image:notfound \
  --restart=Never

sleep 20
kubectl get pod imgpull-fail -n troubleshoot-demo
```

**预期输出**：状态为 `ImagePullBackOff` 或 `ErrImagePull`

```bash
# 定位
kubectl describe pod imgpull-fail -n troubleshoot-demo | grep -A10 "Events:"
```

**预期输出**：Events 中显示 `Failed to pull image` 和 `Back-off pulling image` 事件

```bash
# 修复：删除错误 Pod，改用正确镜像
kubectl delete pod imgpull-fail -n troubleshoot-demo
kubectl run imgpull-ok -n troubleshoot-demo \
  --image=public.ecr.aws/nginx/nginx:latest \
  --restart=Never
kubectl wait pod imgpull-ok -n troubleshoot-demo --for=condition=Ready --timeout=60s
```

**预期输出**：`pod/imgpull-ok condition met`

### 6. 制造并排查 CrashLoopBackOff

```bash
# 制造：容器立即退出
kubectl apply -n troubleshoot-demo -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: crash-loop
spec:
  containers:
  - name: app
    image: public.ecr.aws/docker/library/busybox:latest
    command: ["/bin/sh", "-c", "echo 'starting'; exit 1"]
EOF

sleep 30
kubectl get pod crash-loop -n troubleshoot-demo
```

**预期输出**：状态为 `CrashLoopBackOff`

```bash
# 定位
kubectl logs crash-loop -n troubleshoot-demo --previous
kubectl describe pod crash-loop -n troubleshoot-demo | grep -E "Exit Code|Reason|Message"
```

**预期输出**：logs 显示 `starting`；describe 显示 `Exit Code: 1`

### 7. 制造并排查 OOMKilled

```bash
# 制造：Python 分配超过内存 limit 的内存
kubectl apply -n troubleshoot-demo -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: oom-kill
spec:
  containers:
  - name: app
    image: public.ecr.aws/docker/library/python:3.12-alpine
    command: ["python3", "-c", "x = ' ' * 100 * 1024 * 1024; import time; time.sleep(60)"]
    resources:
      limits:
        memory: "10Mi"
EOF

sleep 15
kubectl get pod oom-kill -n troubleshoot-demo
```

**预期输出**：状态为 `OOMKilled` 或 `Error`

```bash
# 定位
kubectl describe pod oom-kill -n troubleshoot-demo | grep -E "OOMKilled|Exit Code|Reason"
```

**预期输出**：`Reason: OOMKilled` 且 `Exit Code: 137`

> ⚠️ OOMKilled 必须用 Python 分配内存（`x = ' ' * 100 * 1024 * 1024`），使用 `dd` 或 `/dev/urandom` 不会实际分配物理内存，无法触发 OOM。Exit Code 137 = 128 + SIGKILL(9)。

### 8. 制造并排查 Pending（资源不足）

```bash
# 制造：请求超大 CPU
kubectl apply -n troubleshoot-demo -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pending-pod
spec:
  containers:
  - name: app
    image: public.ecr.aws/nginx/nginx:latest
    resources:
      requests:
        cpu: "100"
        memory: "100Gi"
EOF

sleep 10
kubectl get pod pending-pod -n troubleshoot-demo
```

**预期输出**：状态为 `Pending`

```bash
# 定位
kubectl describe pod pending-pod -n troubleshoot-demo | grep -A10 "Events:"
```

**预期输出**：Events 中显示 `Insufficient cpu` 或 `0/3 nodes are available`

### 9. 制造并排查 Service selector 错误（Endpoints 为空）

```bash
kubectl apply -n troubleshoot-demo -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: selector-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: selector-demo
  template:
    metadata:
      labels:
        app: selector-demo
    spec:
      containers:
      - name: nginx
        image: public.ecr.aws/nginx/nginx:latest
---
apiVersion: v1
kind: Service
metadata:
  name: selector-broken-svc
spec:
  selector:
    app: wrong-label
  ports:
  - port: 80
EOF

sleep 15
kubectl get endpoints selector-broken-svc -n troubleshoot-demo
```

**预期输出**：ENDPOINTS 列为 `<none>`（selector 不匹配导致 Endpoints 为空）

```bash
# 修复：更正 selector
kubectl patch svc selector-broken-svc -n troubleshoot-demo \
  -p '{"spec":{"selector":{"app":"selector-demo"}}}'
sleep 5
kubectl get endpoints selector-broken-svc -n troubleshoot-demo
```

**预期输出**：ENDPOINTS 列显示 Pod IP 地址（非 `<none>`）

### 10. 演示 kubectl debug 进入无 shell 容器排障

```bash
# 部署无 shell 容器
kubectl apply -n troubleshoot-demo -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: distroless-pod
spec:
  containers:
  - name: app
    image: gcr.io/distroless/python3-debian12:nonroot
    command: ["python3", "-c", "import time; time.sleep(3600)"]
EOF

sleep 10

# 使用 kubectl debug 附加 ephemeral container 排障
kubectl debug -it distroless-pod -n troubleshoot-demo \
  --image=public.ecr.aws/docker/library/busybox:latest \
  --target=app \
  --profile=general \
  -- sh -c "cat /proc/1/cmdline && echo '' && echo 'debug container 成功进入'"
```

**预期输出**：`debug container 成功进入`

> ⚠️ `kubectl debug` 为 Pod 添加 ephemeral container，可访问目标容器的 proc 文件系统，适用于无 shell 的 distroless 镜像排障。

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
| 1 | `kubectl get deployment probe-demo -n troubleshoot-demo -o jsonpath='{.status.readyReplicas}'` | `1` |
| 2 | `kubectl get pod liveness-fail -n troubleshoot-demo -o jsonpath='{.status.containerStatuses[0].restartCount}' 2>/dev/null \| awk '{if ($1>0) print "restarted"; else print "not-restarted"}'` | `restarted` |
| 3 | `kubectl get pod oom-kill -n troubleshoot-demo -o jsonpath='{.status.containerStatuses[0].lastState.terminated.exitCode}' 2>/dev/null` | `137` |
| 4 | `kubectl get pod pending-pod -n troubleshoot-demo -o jsonpath='{.status.phase}'` | `Pending` |
| 5 | `kubectl get endpoints selector-broken-svc -n troubleshoot-demo -o jsonpath='{.subsets[0].addresses[0].ip}' 2>/dev/null \| grep -c '\.'` | `1` |

---

## 实验总结

本实验系统性地演示了 Kubernetes 三种健康检查探针（startup、readiness、liveness）的工作机制，并通过制造和排查 ImagePullBackOff、CrashLoopBackOff、OOMKilled、Pending、Service selector 错误等常见故障，掌握了 kubectl describe、logs、debug 等核心排障工具的使用方法。理解这些故障模式和排查思路是运维 EKS 集群的基本功。下一个实验将学习使用 Velero 进行集群备份和恢复，为灾难恢复场景做好准备。

---

## 清理

```bash
kubectl delete namespace troubleshoot-demo
echo "清理完成"
```
