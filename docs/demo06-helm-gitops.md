# Demo06 — Helm 与 GitOps 发布

## 实验简介

Helm 是 Kubernetes 的包管理器，通过 Chart 模板化应用部署并支持版本化发布与回滚。Argo CD 是声明式 GitOps 持续交付工具，以 Git 仓库为唯一事实来源，自动将集群状态与期望状态同步。本实验将两者结合，演示从手动 Helm 发布到 GitOps 自动化的完整流程。

**实验目标：**
- 掌握 Helm Chart 的创建、安装、升级和回滚操作
- 部署 Argo CD 并配置 Application 实现自动同步
- 验证 GitOps 自愈（Self-healing）和漂移检测能力
- 理解声明式发布管理相比命令式操作的优势

**实验流程：**
1. 创建本地 Helm Chart 并完成 install → upgrade → rollback 生命周期
2. 安装 Argo CD 并登录管理界面
3. 创建 GitOps Application 并验证自动同步
4. 模拟自愈、漂移检测和手动同步恢复场景

**预计 AI 执行时长：** 5 分钟


## 前提条件

- **工具**：kubectl、Helm、aws CLI、argocd CLI（步骤中安装）
- **权限**：EKS 操作权限
- **前提**：Demo01 集群已创建，Demo02 ALB Controller 已安装
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

## 第一部分：Helm 发布管理

### 1. 创建本地 Helm Chart

```bash
helm create nginx-demo
sed -i 's|repository: nginx|repository: public.ecr.aws/docker/library/nginx|' nginx-demo/values.yaml
sed -i 's|tag: ""|tag: "1.27-alpine"|' nginx-demo/values.yaml
```

**预期输出**：无错误，`nginx-demo/` 目录已创建

### 2. 安装 Chart（install）

```bash
kubectl create namespace helm-demo
helm install nginx-demo ./nginx-demo -n helm-demo
```

**预期输出**：`STATUS: deployed`，`REVISION: 1`

```bash
kubectl rollout status deployment/nginx-demo -n helm-demo --timeout=3m
```

**预期输出**：`deployment "nginx-demo" successfully rolled out`

### 3. 升级 Chart（upgrade）— 扩容到 3 副本

```bash
helm upgrade nginx-demo ./nginx-demo -n helm-demo --set replicaCount=3
kubectl rollout status deployment/nginx-demo -n helm-demo --timeout=3m
helm list -n helm-demo
```

**预期输出**：`REVISION: 2`，`STATUS: deployed`

### 4. 回滚到版本 1（rollback）

```bash
helm rollback nginx-demo 1 -n helm-demo
kubectl rollout status deployment/nginx-demo -n helm-demo --timeout=3m
helm history nginx-demo -n helm-demo
```

**预期输出**：history 显示 3 个 revision，最新 revision 状态为 `deployed`，描述包含 `Rollback to 1`

---

## 第二部分：Argo CD GitOps

### 5. 安装 Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml \
  --server-side

# 等待 argocd-server
kubectl rollout status deployment/argocd-server -n argocd --timeout=5m
# 等待 argocd-application-controller（StatefulSet）
kubectl rollout status statefulset/argocd-application-controller -n argocd --timeout=5m
```

**预期输出**：`deployment "argocd-server" successfully rolled out` 和 `statefulset rolling update complete`

> ⚠️ `argocd-application-controller` 是 StatefulSet，必须用 `kubectl rollout status statefulset/` 等待，不要用 deployment。

### 6. 安装 argocd CLI

```bash
if ! command -v argocd &>/dev/null; then
  curl -sSL -o /tmp/argocd \
    https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
  sudo install -m 755 /tmp/argocd /usr/local/bin/argocd
fi
argocd version --client
```

**预期输出**：输出 argocd 客户端版本号

### 7. 登录 Argo CD

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
ARGOCD_PF_PID=$!
sleep 5

ARGOCD_PASS=$(kubectl get secret argocd-initial-admin-secret \
  -n argocd -o jsonpath='{.data.password}' | base64 -d)
argocd login localhost:8080 \
  --username admin --password ${ARGOCD_PASS} --insecure
echo "Argo CD 登录成功"
```

**预期输出**：`'admin:login' logged in successfully`，`Argo CD 登录成功`

### 8. 创建 Application（自动同步 + 自愈）

```bash
kubectl create namespace gitops-demo

argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace gitops-demo \
  --sync-policy automated \
  --self-heal

argocd app wait guestbook --sync --health --timeout 120
argocd app get guestbook
kubectl get pods -n gitops-demo
```

**预期输出**：Application Health Status 为 `Healthy`，Sync Status 为 `Synced`；pods 有 Running 状态的容器

> ⚠️ CodeCommit HTTPS 须使用 IAM service-specific credentials，标准 IAM Access Key 不可用，Workshop 环境使用 GitHub 公开仓库替代。

### 9. 场景一：自愈（Self-healing）

```bash
echo "=== 删除所有 Pod，观察自愈 ==="
kubectl delete pods -n gitops-demo --all
sleep 30
kubectl get pods -n gitops-demo
argocd app get guestbook | grep -E "Health|Sync"
```

**预期输出**：Pod 被删除后重新创建；Health Status 仍为 `Healthy`，Sync Status 为 `Synced`

### 10. 场景二：漂移检测 + Diff

```bash
# 关闭自动同步，制造漂移
argocd app set guestbook --sync-policy none
kubectl scale deployment guestbook-ui -n gitops-demo --replicas=3
sleep 5
echo "=== 当前状态（应为 OutOfSync）==="
argocd app get guestbook | grep -E "Health|Sync|Status"
echo "=== Diff（期望 Git 状态 vs 实际集群状态）==="
argocd app diff guestbook
```

**预期输出**：Sync Status 为 `OutOfSync`；Diff 显示 replicas 从 1 变为 3

### 11. 场景三：手动同步恢复

```bash
argocd app sync guestbook
argocd app wait guestbook --sync --health --timeout 60
kubectl get pods -n gitops-demo
argocd app history guestbook
```

**预期输出**：Sync Status 为 `Synced`；Health Status 为 `Healthy`；副本数恢复为 1

---

## 验收标准

完成本实验后，你应当能够：
- [ ] 使用 Helm 完成 Chart 的 install、upgrade 和 rollback 生命周期管理
- [ ] 部署 Argo CD 并通过 CLI 创建自动同步的 GitOps Application
- [ ] 验证 GitOps 自愈能力和漂移检测机制

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `helm history nginx-demo -n helm-demo -o json \| python3 -c "import sys,json; print(len(json.load(sys.stdin)))"` | `3` |
| 2 | `helm list -n helm-demo -o json \| python3 -c "import sys,json; print(json.load(sys.stdin)[0]['status'])"` | `deployed` |
| 3 | `kubectl get deployment argocd-server -n argocd -o jsonpath='{.status.readyReplicas}'` | `1` |
| 4 | `argocd app get guestbook -o json 2>/dev/null \| python3 -c "import sys,json; d=json.load(sys.stdin); print(d['status']['sync']['status'])"` | `Synced` |
| 5 | `argocd app get guestbook -o json 2>/dev/null \| python3 -c "import sys,json; d=json.load(sys.stdin); print(d['status']['health']['status'])"` | `Healthy` |

---

## 实验总结

本实验演示了从手动 Helm 发布到 Argo CD GitOps 自动化的完整流程。Helm 提供了版本化的发布管理和一键回滚能力，而 Argo CD 以 Git 为唯一事实来源实现了声明式部署、自愈和漂移检测。两者结合是生产环境中最常见的发布管理模式，后续实验将在此基础上构建更复杂的应用部署场景。

---

## 清理

```bash
# 停止 port-forward
kill ${ARGOCD_PF_PID} 2>/dev/null || pkill -f "kubectl port-forward svc/argocd-server" || true

# 删除 Argo CD Application
argocd app delete guestbook --yes 2>/dev/null || true

# 删除 namespace
kubectl delete namespace argocd gitops-demo helm-demo

# 清理本地 Helm chart 文件
rm -rf ./nginx-demo

echo "清理完成"
```
