# Demo16 — 使用 NetworkPolicy 限制 Pod 流量

## 实验简介

本实验将完成「使用 NetworkPolicy 限制 Pod 流量」的全部操作，涵盖资源创建、配置、验证的完整流程。

**实验目标：**
- 掌握 使用 NetworkPolicy 限制 Pod 流量 的核心操作流程
- 理解相关组件的工作原理和配置要点
- 能够独立完成从部署到验证的完整闭环

**实验流程：**
1. 启用 VPC CNI NetworkPolicy
2. 部署 backend 和两个客户端
3. 策略前基准测试（两个客户端均可访问）
4. 应用 Ingress NetworkPolicy
5. 验证 Ingress 策略效果
6. 创建 Default-deny namespace
7. 按需放开 client → server 流量
8. 限制 client-allowed 只能访问 backend

**预计 AI 执行时长：** 8-10 分钟


## 前提条件

- **工具**：AWS CLI v2、eksctl、kubectl v1.35
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

## 步骤

### 1. 启用 VPC CNI NetworkPolicy

```bash
aws eks update-addon \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name vpc-cni \
  --configuration-values '{"enableNetworkPolicy": "true"}' \
  --region ${AWS_REGION} \
  --resolve-conflicts OVERWRITE

until [ "$(aws eks describe-addon --cluster-name ${CLUSTER_NAME} \
  --addon-name vpc-cni --region ${AWS_REGION} \
  --query 'addon.status' --output text)" = "ACTIVE" ]; do
  echo "等待 vpc-cni addon ACTIVE..."
  sleep 10
done
echo "vpc-cni NetworkPolicy 已启用"
```

**预期输出**：`vpc-cni NetworkPolicy 已启用`

> ⚠️ VPC CNI v1.19+ 中 NetworkPolicy agent 已合并进 `aws-eks-nodeagent` 容器，不再单独显示为 `aws-network-policy-agent`。

---

## 第一部分：Ingress 策略（基于标签选择器）

### 2. 部署 backend 和两个客户端

```bash
kubectl create namespace netpol-demo

kubectl apply -n netpol-demo -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
  - port: 80
EOF

kubectl run client-allowed -n netpol-demo \
  --image=curlimages/curl:latest \
  --labels="role=allowed" --restart=Never -- sleep 3600
kubectl run client-blocked -n netpol-demo \
  --image=curlimages/curl:latest \
  --labels="role=blocked" --restart=Never -- sleep 3600

kubectl wait --for=condition=Ready pod/client-allowed pod/client-blocked \
  -n netpol-demo --timeout=2m
echo "Pod 就绪"
```

**预期输出**：打印"Pod 就绪"

### 3. 策略前基准测试（两个客户端均可访问）

```bash
echo "=== 策略前基准测试 ==="
kubectl exec -n netpol-demo client-allowed -- \
  curl -s -o /dev/null -w "allowed→backend: %{http_code}\n" --max-time 5 http://backend/
kubectl exec -n netpol-demo client-blocked -- \
  curl -s -o /dev/null -w "blocked→backend: %{http_code}\n" --max-time 5 http://backend/
```

**预期输出**：两个客户端均返回 `200`

### 4. 应用 Ingress NetworkPolicy

```bash
kubectl apply -n netpol-demo -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-ingress
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: allowed
    ports:
    - protocol: TCP
      port: 80
EOF

echo "等待策略生效（10 秒）..."
sleep 10
```

**预期输出**：打印"等待策略生效"

> ⚠️ NetworkPolicy apply 后需 sleep 10 再验证，立即测试可能因规则尚未下发而仍放行。

### 5. 验证 Ingress 策略效果

```bash
echo "=== Ingress 策略后验证 ==="
kubectl exec -n netpol-demo client-allowed -- \
  curl -s -o /dev/null -w "allowed→backend: %{http_code}\n" --max-time 5 http://backend/
kubectl exec -n netpol-demo client-blocked -- \
  curl -s -o /dev/null -w "blocked→backend(expect timeout): %{http_code}\n" --max-time 5 http://backend/ || echo "blocked: 已超时"
```

**预期输出**：`allowed→backend: 200`；`client-blocked` 超时（exit 28）

---

## 第二部分：Default-deny 模式（生产推荐）

### 6. 创建 Default-deny namespace

```bash
kubectl create namespace netpol-isolated

kubectl apply -n netpol-isolated -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF

kubectl run server -n netpol-isolated \
  --image=nginx:alpine --labels="app=server" --restart=Never --port=80
kubectl run client -n netpol-isolated \
  --image=curlimages/curl:latest --labels="app=client" --restart=Never -- sleep 3600

kubectl wait --for=condition=Ready pod/server pod/client \
  -n netpol-isolated --timeout=2m

SERVER_IP=$(kubectl get pod server -n netpol-isolated -o jsonpath='{.status.podIP}')
echo "Server IP: ${SERVER_IP}"

sleep 5
echo "=== Default-deny 后（均应超时）==="
kubectl exec -n netpol-isolated client -- \
  curl -s -o /dev/null -w "exit=%{exitcode}\n" --max-time 5 http://${SERVER_IP}/ || true
```

**预期输出**：curl 超时，exit code 为 28

### 7. 按需放开 client → server 流量

```bash
kubectl apply -n netpol-isolated -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-client-to-server
spec:
  podSelector:
    matchLabels:
      app: server
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: client
    ports:
    - protocol: TCP
      port: 80
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-client-egress-to-server
spec:
  podSelector:
    matchLabels:
      app: client
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: server
    ports:
    - protocol: TCP
      port: 80
  - ports:
    - protocol: UDP
      port: 53
EOF

sleep 10
SERVER_IP=$(kubectl get pod server -n netpol-isolated -o jsonpath='{.status.podIP}')
echo "=== 按需放开后 ==="
kubectl exec -n netpol-isolated client -- \
  curl -s -o /dev/null -w "client→server: %{http_code}\n" --max-time 5 http://${SERVER_IP}/
```

**预期输出**：`client→server: 200`

---

## 第三部分：Egress 策略（禁止 Pod 出公网）

### 8. 限制 client-allowed 只能访问 backend

```bash
kubectl apply -n netpol-demo -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: client-egress-restrict
spec:
  podSelector:
    matchLabels:
      role: allowed
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 80
  - ports:
    - protocol: UDP
      port: 53
EOF

sleep 10
echo "=== Egress 限制后 ==="
echo -n "client-allowed→backend(expect 200): "
kubectl exec -n netpol-demo client-allowed -- \
  curl -s -o /dev/null -w "%{http_code}\n" --max-time 5 http://backend/
echo -n "client-allowed→公网(expect timeout): "
kubectl exec -n netpol-demo client-allowed -- \
  curl -s -o /dev/null -w "exit=%{exitcode}\n" --max-time 5 http://1.1.1.1/ || true
```

**预期输出**：访问 backend 返回 `200`；访问公网超时（exit 28）

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
| 1 | `aws eks describe-addon --cluster-name demo --addon-name vpc-cni --region us-east-1 --query 'addon.status' --output text` | `ACTIVE` |
| 2 | `kubectl get networkpolicy -n netpol-demo --no-headers \| wc -l \| tr -d ' '` | `2` |
| 3 | `kubectl get namespace netpol-isolated -o jsonpath='{.status.phase}'` | `Active` |
| 4 | `kubectl get networkpolicy default-deny-all -n netpol-isolated -o jsonpath='{.spec.podSelector}'` | `{}` |

---

## 实验总结

本实验完成了「使用 NetworkPolicy 限制 Pod 流量」的全部操作，从资源创建到功能验证形成了完整闭环。通过动手实践，你已掌握了相关组件的核心配置方法和最佳实践。Demo17 将学习 Pod Security Standards 工作负载安全。

---

## 清理

```bash
kubectl delete namespace netpol-demo netpol-isolated 2>/dev/null || true

aws eks update-addon \
  --cluster-name ${CLUSTER_NAME} \
  --addon-name vpc-cni \
  --configuration-values '{"enableNetworkPolicy": "false"}' \
  --region ${AWS_REGION} \
  --resolve-conflicts OVERWRITE

until [ "$(aws eks describe-addon --cluster-name ${CLUSTER_NAME} \
  --addon-name vpc-cni --region ${AWS_REGION} \
  --query 'addon.status' --output text)" = "ACTIVE" ]; do
  echo "等待 vpc-cni 更新..."
  sleep 10
done
echo "清理完成，vpc-cni NetworkPolicy 已恢复为 false"
```
