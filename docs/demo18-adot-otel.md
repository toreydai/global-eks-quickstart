# Demo18 — ADOT 与 OpenTelemetry 可观测性

## 实验简介

本实验将完成「ADOT 与 OpenTelemetry 可观测性」的全部操作，涵盖资源创建、配置、验证的完整流程。

**实验目标：**
- 掌握 ADOT 与 OpenTelemetry 可观测性 的核心操作流程
- 理解相关组件的工作原理和配置要点
- 能够独立完成从部署到验证的完整闭环

**实验流程：**
1. 创建 ADOT Collector IAM Policy
2. 创建 Pod Identity IAM Role
3. 部署 ADOT Collector
4. 发送 OTLP Trace 数据
5. 验证 ADOT Collector 日志
6. 验证 X-Ray trace
7. 验证 CloudWatch Metrics

**预计 AI 执行时长：** 5-8 分钟


## 前提条件

- **工具**：AWS CLI v2、eksctl、kubectl v1.35、Python 3
- **权限**：AdministratorAccess（含 X-Ray 和 CloudWatch 写权限）
- **前提**：Demo01 已完成，eks-pod-identity-agent 插件已安装
- **初始化**：

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
export CLUSTER_NAME=demo
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ADOT_NS=adot
```

---

## 步骤

### 1. 创建 ADOT Collector IAM Policy

```bash
cat > /tmp/adot-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "xray:PutTraceSegments",
        "xray:PutTelemetryRecords",
        "xray:GetSamplingRules",
        "xray:GetSamplingTargets",
        "xray:GetSamplingStatisticSummaries"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricData"
      ],
      "Resource": "*"
    }
  ]
}
EOF

ADOT_POLICY_ARN=$(aws iam create-policy \
  --policy-name ADOT-Collector-Policy \
  --policy-document file:///tmp/adot-policy.json \
  --query Policy.Arn --output text)

echo "ADOT Policy ARN: ${ADOT_POLICY_ARN}"
```

**预期输出**：打印 Policy ARN

### 2. 创建 Pod Identity IAM Role

```bash
cat > /tmp/pod-trust.json << 'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"pods.eks.amazonaws.com"},"Action":["sts:AssumeRole","sts:TagSession"]}]}
EOF

ADOT_ROLE_ARN=$(aws iam create-role \
  --role-name ADOT-Collector-Role \
  --assume-role-policy-document file:///tmp/pod-trust.json \
  --query Role.Arn --output text)

aws iam attach-role-policy \
  --role-name ADOT-Collector-Role \
  --policy-arn ${ADOT_POLICY_ARN}

kubectl create namespace ${ADOT_NS}

aws eks create-pod-identity-association \
  --cluster-name ${CLUSTER_NAME} \
  --namespace ${ADOT_NS} \
  --service-account adot-collector \
  --role-arn ${ADOT_ROLE_ARN} \
  --region ${AWS_REGION}

echo "Pod Identity 关联创建完成"
```

**预期输出**：打印"Pod Identity 关联创建完成"

### 3. 部署 ADOT Collector

创建 ConfigMap（使用 `debug` exporter，`logging` 已废弃）：

```bash
kubectl apply -n ${ADOT_NS} -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: adot-config
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

    exporters:
      awsxray:
        region: ${AWS_REGION}
      awsemf:
        region: ${AWS_REGION}
        namespace: EKS/ADOT/Demo
        log_group_name: /eks/adot/demo
        log_stream_name: demo18-stream
      debug:
        verbosity: detailed

    service:
      pipelines:
        traces:
          receivers: [otlp]
          exporters: [awsxray, debug]
        metrics:
          receivers: [otlp]
          exporters: [awsemf, debug]
EOF
```

**预期输出**：ConfigMap 创建成功

> ⚠️ 必须使用 `debug` exporter，`logging` exporter 已在 ADOT v0.48.0 废弃。CloudWatch 指标使用 `awsemf` exporter（非 `awscloudwatch`）。

```bash
kubectl create serviceaccount adot-collector -n ${ADOT_NS} 2>/dev/null || true

kubectl apply -n ${ADOT_NS} -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: adot-collector
spec:
  replicas: 1
  selector:
    matchLabels:
      app: adot-collector
  template:
    metadata:
      labels:
        app: adot-collector
    spec:
      serviceAccountName: adot-collector
      containers:
      - name: adot-collector
        image: public.ecr.aws/aws-observability/aws-otel-collector:latest
        args: ["--config=/conf/config.yaml"]
        ports:
        - containerPort: 4317
          name: otlp-grpc
        - containerPort: 4318
          name: otlp-http
        volumeMounts:
        - name: config
          mountPath: /conf
      volumes:
      - name: config
        configMap:
          name: adot-config
---
apiVersion: v1
kind: Service
metadata:
  name: adot-collector
spec:
  selector:
    app: adot-collector
  ports:
  - name: otlp-grpc
    port: 4317
    targetPort: 4317
  - name: otlp-http
    port: 4318
    targetPort: 4318
EOF

kubectl rollout status deployment/adot-collector -n ${ADOT_NS} --timeout=5m
echo "ADOT Collector 已部署"
```

**预期输出**：`deployment "adot-collector" successfully rolled out`

### 4. 发送 OTLP Trace 数据

部署 Python 客户端发送 trace（注意 trace ID 格式：前 8 位为当前 Unix 时间戳）：

```bash
kubectl run trace-sender -n ${ADOT_NS} \
  --image=public.ecr.aws/docker/library/python:3.12-alpine \
  --restart=Never -- sleep 3600

kubectl wait --for=condition=Ready pod/trace-sender -n ${ADOT_NS} --timeout=2m

kubectl exec -n ${ADOT_NS} trace-sender -- python3 -c "
import urllib.request, json, time, random

ADOT_URL = 'http://adot-collector.adot.svc.cluster.local:4318/v1/traces'
epoch_hex = '%08x' % int(time.time())
random_hex = '%024x' % random.getrandbits(96)
trace_id = epoch_hex + random_hex
span_id = '%016x' % random.getrandbits(64)

payload = {
  'resourceSpans': [{
    'resource': {'attributes': [{'key': 'service.name', 'value': {'stringValue': 'demo18-test'}}]},
    'scopeSpans': [{
      'spans': [{
        'traceId': trace_id,
        'spanId': span_id,
        'name': 'demo18-test-span',
        'kind': 1,
        'startTimeUnixNano': str(int(time.time() * 1e9)),
        'endTimeUnixNano': str(int((time.time() + 0.1) * 1e9)),
        'status': {'code': 1}
      }]
    }]
  }]
}

data = json.dumps(payload).encode()
req = urllib.request.Request(ADOT_URL, data=data,
  headers={'Content-Type': 'application/json'}, method='POST')
resp = urllib.request.urlopen(req, timeout=10)
print(f'HTTP {resp.status}: trace_id={trace_id[:8]}...')
"
```

**预期输出**：`HTTP 200: trace_id=xxxxxxxx...`

> ⚠️ OTLP trace ID 前 8 位必须为当前 Unix 时间戳的 hex 编码，否则 X-Ray 因时间戳偏差丢弃 segment。使用 HTTP 4318 端口（gRPC 4317 需 HTTP/2，Python urllib 不支持）。

### 5. 验证 ADOT Collector 日志

```bash
kubectl logs deployment/adot-collector -n ${ADOT_NS} | tail -20
```

**预期输出**：日志中出现 `demo18-test-span` 的 span 信息

### 6. 验证 X-Ray trace

```bash
sleep 10
START_TIME=$(date -d '5 minutes ago' --utc +%s 2>/dev/null || date -v-5M +%s)
END_TIME=$(date --utc +%s 2>/dev/null || date +%s)

aws xray get-trace-summaries \
  --start-time ${START_TIME} \
  --end-time ${END_TIME} \
  --region ${AWS_REGION} \
  --query 'length(TraceSummaries)' \
  --output text
```

**预期输出**：大于 0 的数字（有 trace 记录）

> ⚠️ 若返回 0，检查 Collector 日志中是否有 `UnprocessedTraceSegments:[]`（表示 API 调用成功但 X-Ray 索引延迟）。首次使用 X-Ray 的账号可能需先访问 AWS X-Ray 控制台激活索引功能。

### 7. 验证 CloudWatch Metrics

```bash
sleep 30
aws cloudwatch list-metrics \
  --namespace "EKS/ADOT/Demo" \
  --region ${AWS_REGION} \
  --query 'length(Metrics)' \
  --output text
```

**预期输出**：大于等于 0（metrics 需要一定时间才出现在 CloudWatch）

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
| 1 | `kubectl get deployment adot-collector -n adot -o jsonpath='{.status.readyReplicas}'` | `1` |
| 2 | `kubectl get pods -n adot --no-headers \| grep Running \| wc -l \| tr -d ' '` | `2` |
| 3 | `aws logs describe-log-groups --log-group-name-prefix "/eks/adot" --region us-east-1 --query 'length(logGroups)' --output text` | `1` |

---

## 实验总结

本实验完成了「ADOT 与 OpenTelemetry 可观测性」的全部操作，从资源创建到功能验证形成了完整闭环。通过动手实践，你已掌握了相关组件的核心配置方法和最佳实践。至此完成 EKS QuickStart 全部实验，可进入 global-eks-advance 进阶课程。

---

## 清理

```bash
kubectl delete pod trace-sender -n ${ADOT_NS} 2>/dev/null || true
kubectl delete -n ${ADOT_NS} deployment/adot-collector service/adot-collector configmap/adot-config serviceaccount/adot-collector 2>/dev/null || true
kubectl delete namespace ${ADOT_NS} 2>/dev/null || true

for ASSOC in $(aws eks list-pod-identity-associations --cluster-name ${CLUSTER_NAME} \
  --namespace adot --service-account adot-collector \
  --query 'associations[*].associationId' --output text 2>/dev/null); do
  aws eks delete-pod-identity-association --cluster-name ${CLUSTER_NAME} \
    --association-id ${ASSOC} --region ${AWS_REGION}
done

aws iam detach-role-policy \
  --role-name ADOT-Collector-Role \
  --policy-arn ${ADOT_POLICY_ARN} 2>/dev/null || true
aws iam delete-role --role-name ADOT-Collector-Role 2>/dev/null || true
aws iam delete-policy --policy-arn ${ADOT_POLICY_ARN} 2>/dev/null || true

for LG in $(aws logs describe-log-groups \
  --log-group-name-prefix "/eks/adot" \
  --region ${AWS_REGION} \
  --query 'logGroups[*].logGroupName' --output text 2>/dev/null); do
  aws logs delete-log-group --log-group-name "${LG}" --region ${AWS_REGION}
  echo "已删除日志组: ${LG}"
done

echo "清理完成"
```
