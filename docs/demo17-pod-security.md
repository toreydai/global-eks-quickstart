# Demo17 — Pod Security Standards 工作负载安全

## 实验简介

本实验将完成「Pod Security Standards 工作负载安全」的全部操作，涵盖资源创建、配置、验证的完整流程。

**实验目标：**
- 掌握 Pod Security Standards 工作负载安全 的核心操作流程
- 理解相关组件的工作原理和配置要点
- 能够独立完成从部署到验证的完整闭环

**实验流程：**
1. 创建三个不同安全等级的 namespace
2. 测试特权 Pod 在三个 namespace 中的行为
3. 测试普通 nginx Pod 在 baseline 和 restricted 中的行为
4. 创建 warn + audit namespace
5. 部署普通 nginx（触发 warn 但不阻止创建）
6. 创建完全合规的 restricted Pod
7. 验证合规 Pod 运行身份

**预计 AI 执行时长：** 3 分钟


## 前提条件

- **工具**：AWS CLI v2、kubectl v1.35
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

## 背景：PSS 三个等级

| 等级 | 特点 | 适用场景 |
|------|------|---------|
| privileged | 无限制 | 系统组件、受信任工作负载 |
| baseline | 禁止明显危险配置（privileged、hostNetwork 等）| 通用应用 |
| restricted | 最严格，要求非 root、drop ALL、seccompProfile | 安全敏感应用 |

---

## 步骤

## 第一部分：三个 PSS 等级对比

### 1. 创建三个不同安全等级的 namespace

```bash
kubectl create namespace pss-privileged
kubectl label namespace pss-privileged \
  pod-security.kubernetes.io/enforce=privileged

kubectl create namespace pss-baseline
kubectl label namespace pss-baseline \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest

kubectl create namespace pss-restricted
kubectl label namespace pss-restricted \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest

kubectl get namespace pss-privileged pss-baseline pss-restricted \
  --show-labels | grep pod-security
```

**预期输出**：三个 namespace 均显示对应的 `pod-security.kubernetes.io/enforce` label

### 2. 测试特权 Pod 在三个 namespace 中的行为

```bash
PRIV_POD='{"apiVersion":"v1","kind":"Pod","metadata":{"name":"test-pod"},"spec":{"containers":[{"name":"c","image":"nginx:alpine","securityContext":{"privileged":true}}]}}'

echo "=== privileged namespace（expect: 创建成功）==="
echo $PRIV_POD | kubectl apply -n pss-privileged -f - 2>&1

echo "=== baseline namespace（expect: Forbidden - privileged 违规）==="
echo $PRIV_POD | kubectl apply -n pss-baseline -f - 2>&1

echo "=== restricted namespace（expect: Forbidden - 多项违规）==="
echo $PRIV_POD | kubectl apply -n pss-restricted -f - 2>&1
```

**预期输出**：
- `pss-privileged`：Pod 创建成功
- `pss-baseline` 和 `pss-restricted`：`Error from server (Forbidden): pods "test-pod" is forbidden`

### 3. 测试普通 nginx Pod 在 baseline 和 restricted 中的行为

```bash
NGINX_POD='{"apiVersion":"v1","kind":"Pod","metadata":{"name":"nginx-pod"},"spec":{"containers":[{"name":"c","image":"nginx:alpine"}]}}'

echo "=== baseline namespace（expect: 创建成功，nginx root 不违反 baseline）==="
echo $NGINX_POD | kubectl apply -n pss-baseline -f - 2>&1

echo "=== restricted namespace（expect: Forbidden - runAsNonRoot/seccompProfile 违规）==="
echo $NGINX_POD | kubectl apply -n pss-restricted -f - 2>&1
```

**预期输出**：
- `pss-baseline`：Pod 创建成功
- `pss-restricted`：`Error from server (Forbidden)`（违反 allowPrivilegeEscalation、capabilities.drop、runAsNonRoot、seccompProfile 要求）

---

## 第二部分：warn + audit 模式

### 4. 创建 warn + audit namespace

```bash
kubectl create namespace pss-warn-demo
kubectl label namespace pss-warn-demo \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/warn-version=latest \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/audit-version=latest

echo "namespace 已配置：enforce=baseline, warn=restricted, audit=restricted"
```

**预期输出**：打印配置确认信息

### 5. 部署普通 nginx（触发 warn 但不阻止创建）

```bash
kubectl apply -n pss-warn-demo -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: nginx-warn
spec:
  containers:
  - name: nginx
    image: nginx:alpine
EOF
```

**预期输出**：kubectl 打印 `Warning: ...restricted...`（来自 warn=restricted），同时 Pod 实际被创建

```bash
kubectl get pod nginx-warn -n pss-warn-demo
```

**预期输出**：Pod 状态为 `Running`（enforce=baseline 允许创建）

---

## 第三部分：enforce=restricted 下的合规 Pod

### 6. 创建完全合规的 restricted Pod

```bash
kubectl apply -n pss-restricted -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: compliant-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: busybox:latest
    command: ["sleep", "3600"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
EOF

kubectl wait --for=condition=Ready pod/compliant-pod -n pss-restricted --timeout=60s
echo "合规 Pod 创建成功"
```

**预期输出**：打印"合规 Pod 创建成功"

> ⚠️ restricted 等级要求同时满足：`runAsNonRoot`、指定 `seccompProfile`、`capabilities.drop: ["ALL"]`，三者缺一即被拒绝。

### 7. 验证合规 Pod 运行身份

```bash
echo "=== 合规 Pod 运行身份（应为 uid=1000）==="
kubectl exec -n pss-restricted compliant-pod -- id

echo "=== 确认无法写入系统目录（权限拒绝）==="
kubectl exec -n pss-restricted compliant-pod -- sh -c 'touch /etc/test 2>&1 || echo "write blocked: OK"'
```

**预期输出**：`uid=1000`；`write blocked: OK`

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
| 1 | `kubectl get namespace pss-restricted -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/enforce}'` | `restricted` |
| 2 | `kubectl get pod compliant-pod -n pss-restricted -o jsonpath='{.status.phase}'` | `Running` |
| 3 | `kubectl exec -n pss-restricted compliant-pod -- id \| grep -o 'uid=[0-9]*' \| head -1` | `uid=1000` |
| 4 | `kubectl get pod test-pod -n pss-privileged -o jsonpath='{.status.phase}'` | `Running` |

---

## 实验总结

本实验完成了「Pod Security Standards 工作负载安全」的全部操作，从资源创建到功能验证形成了完整闭环。通过动手实践，你已掌握了相关组件的核心配置方法和最佳实践。Demo18 将学习 ADOT 与 OpenTelemetry 分布式追踪。

---

## 清理

```bash
kubectl delete namespace pss-privileged pss-baseline pss-restricted pss-warn-demo 2>/dev/null || true
echo "清理完成"
```
