# Demo13 — Prometheus 与 Grafana 集群监控

## 实验简介

Prometheus 是 Kubernetes 生态中最主流的开源监控方案，负责采集和存储集群指标；Grafana 提供可视化仪表盘，将指标数据转化为可操作的洞察。本实验将在 EKS 集群上部署完整的 Prometheus + Grafana 监控栈，并导入社区 Dashboard 实现集群和 Pod 级别的实时监控。

**实验目标：**
- 掌握使用 Helm 部署 Prometheus 和 Grafana 的标准流程
- 理解 Prometheus 数据采集架构与 Grafana 数据源配置
- 能够独立完成监控 Dashboard 的导入与指标查询

**实验流程：**
1. 确认 EBS CSI Driver 并创建 StorageClass
2. 使用 Helm 安装 Prometheus 和 Grafana
3. 获取访问地址并验证服务运行状态
4. 在浏览器中导入社区 Dashboard 验证监控数据

**预计 AI 执行时长：** 12-15 分钟

---

## 前提条件

- **工具**：AWS CLI v2、eksctl、kubectl v1.35、Helm
- **权限**：AdministratorAccess
- **前提**：Demo01 已完成；EBS CSI Driver 已安装（Demo11 完成）或在本 Demo 中安装
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export CLUSTER_NAME=demo
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

---

## 步骤

### 1. 确认 EBS CSI Driver 已安装

```bash
STATUS=$(aws eks describe-addon --cluster-name ${CLUSTER_NAME} \
  --addon-name aws-ebs-csi-driver --region ${AWS_REGION} \
  --query 'addon.status' --output text 2>/dev/null)

if [ "${STATUS}" != "ACTIVE" ]; then
  echo "EBS CSI Driver 未安装，请先完成 Demo11 第一部分，或手动安装"
  echo "当前状态: ${STATUS}"
else
  echo "EBS CSI Driver 状态: ACTIVE"
fi
```

**预期输出**：`EBS CSI Driver 状态: ACTIVE`

### 2. 创建 ebs-sc StorageClass

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
EOF

kubectl get storageclass ebs-sc
```

**预期输出**：显示 `ebs-sc` StorageClass（PROVISIONER 为 `ebs.csi.aws.com`）

### 3. 安装 Prometheus

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install prometheus prometheus-community/prometheus \
  --namespace monitoring --create-namespace \
  --set alertmanager.persistentVolume.storageClass=ebs-sc \
  --set server.persistentVolume.storageClass=ebs-sc \
  --wait --timeout=10m

echo "Prometheus 已安装"
```

**预期输出**：打印"Prometheus 已安装"

### 4. 检查 alertmanager PVC storageClass（必须）

```bash
kubectl get pvc -n monitoring
```

**预期输出**：所有 PVC 的 STORAGECLASS 列均为 `ebs-sc`

> ⚠️ StatefulSet 的 VolumeClaimTemplate 不会随 `helm upgrade --set` 更新已有 PVC。若 `storage-prometheus-alertmanager-0` 的 STORAGECLASS 列为空，执行以下修复：
> ```bash
> kubectl patch pvc storage-prometheus-alertmanager-0 -n monitoring \
>   -p '{"spec":{"storageClassName":"ebs-sc"}}'
> kubectl delete pod prometheus-alertmanager-0 -n monitoring
> ```

### 5. 安装 Grafana

```bash
helm repo add grafana https://grafana.github.io/helm-charts

helm upgrade --install grafana grafana/grafana \
  --namespace monitoring \
  --set adminPassword=admin123 \
  --set persistence.enabled=true \
  --set persistence.storageClassName=ebs-sc \
  --set service.type=NodePort \
  --set "datasources.datasources\.yaml.apiVersion=1" \
  --set "datasources.datasources\.yaml.datasources[0].name=Prometheus" \
  --set "datasources.datasources\.yaml.datasources[0].type=prometheus" \
  --set "datasources.datasources\.yaml.datasources[0].url=http://prometheus-server.monitoring.svc.cluster.local" \
  --set "datasources.datasources\.yaml.datasources[0].isDefault=true" \
  --wait --timeout=10m

echo "Grafana 已安装"
```

**预期输出**：打印"Grafana 已安装"

### 6. 获取访问地址

```bash
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')
GRAFANA_PORT=$(kubectl get svc grafana -n monitoring -o jsonpath='{.spec.ports[0].nodePort}')
PROM_PORT=$(kubectl get svc prometheus-server -n monitoring -o jsonpath='{.spec.ports[0].nodePort}' 2>/dev/null || echo "N/A")

echo "Grafana:    http://${NODE_IP}:${GRAFANA_PORT}  (admin / admin123)"
echo "Prometheus: http://${NODE_IP}:${PROM_PORT}/graph"
```

**预期输出**：打印完整访问地址

> ⚠️ 节点安全组需开放 NodePort 端口范围（30000–32767）。若无法访问，可将 Grafana service.type 改为 LoadBalancer。

### 7. 验证所有 Pod 运行

```bash
kubectl get pods -n monitoring
kubectl get pvc -n monitoring
```

**预期输出**：所有 Pod 状态为 `Running`，PVC 状态为 `Bound`

### 8. 在浏览器验证 Grafana

在浏览器访问步骤 6 输出的 Grafana 地址，使用 `admin / admin123` 登录。

依次操作：
1. 左侧菜单 → **Connections** → **Data sources** → 确认 Prometheus 状态为绿色
2. 左侧菜单 → **Dashboards** → **Import** → 输入 ID `3119` → 选择 Prometheus datasource → Import
3. 再次 Import Dashboard ID `6417`

**预期结果**：Dashboard 3119（集群监控）和 6417（Pod 监控）显示图表数据

> ⚠️ Dashboard 导入需在浏览器中手动操作，无法通过命令行自动化。

---

## 验收标准

完成本实验后，你应当能够：
- [ ] Prometheus Server 和 Grafana 均处于 Running 状态且 PVC 为 Bound
- [ ] 通过 NodePort 访问 Grafana 并成功登录
- [ ] Grafana 中 Prometheus 数据源连接正常（绿色状态）
- [ ] 导入的 Dashboard（3119/6417）能显示集群和 Pod 监控图表

---

## 验证检查点

| # | 检查命令 | 期望精确输出 |
|---|---------|-------------|
| 1 | `kubectl get deployment prometheus-server -n monitoring -o jsonpath='{.status.readyReplicas}'` | `1` |
| 2 | `kubectl get deployment grafana -n monitoring -o jsonpath='{.status.readyReplicas}'` | `1` |
| 3 | `kubectl get pvc -n monitoring --no-headers \| awk '{print $2}' \| sort -u` | `Bound` |
| 4 | `kubectl get svc grafana -n monitoring -o jsonpath='{.spec.type}'` | `NodePort` |

---

## 实验总结

本实验部署了完整的 Prometheus + Grafana 开源监控栈，实现了集群指标的自动采集、持久化存储和可视化展示。核心概念包括 Prometheus 的 pull-based 采集模型、StorageClass 支撑的持久化卷、以及 Grafana 数据源与 Dashboard 的配置方式。下一步 Demo14 将探索 AWS 原生的 CloudWatch Observability 方案，对比开源与托管监控的差异。

---

## 清理

```bash
helm uninstall grafana -n monitoring 2>/dev/null || true
helm uninstall prometheus -n monitoring 2>/dev/null || true

kubectl delete pvc --all -n monitoring 2>/dev/null || true
kubectl delete namespace monitoring 2>/dev/null || true
kubectl delete storageclass ebs-sc 2>/dev/null || true

echo "清理完成"
```
