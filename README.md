# Global EKS QuickStart
## AWS EKS Hands-on Lab Collection

基于 EKS 1.35 · Amazon Linux 2023 · 全球区（us-east-1）· Prompt 驱动执行

---

## Demo 列表

建议先完成 **基础**，再按角色和时间选择 **进阶**。基础覆盖 EKS 从建集群、部署应用、镜像发布、健康检查、调度伸缩、存储、身份到原生可观测性的最小闭环；进阶用于补充生产专题、平台治理和专项运维能力。

## 基础

### 1. 基础环境

- [Demo01 — 准备实验环境与创建集群](docs/demo01-cluster-setup.md)

### 2. 应用部署、镜像与健康检查

- [Demo02 — 部署 2048 应用与 LBController](docs/demo02-alb-2048.md)
- [Demo03 — ECR 私有镜像与镜像发布](docs/demo03-ecr-images.md)（跨区域/跨账号段落选做）
- [Demo04 — 健康检查与故障排查](docs/demo04-health-troubleshoot.md)

### 3. 发布与调度

- [Demo06 — Helm 与 GitOps 发布](docs/demo06-helm-gitops.md)
- [Demo07 — 调度策略与资源治理](docs/demo07-scheduling.md)
- [Demo08 — 使用 HPA 进行自动伸缩](docs/demo08-hpa.md)
- [Demo09 — Karpenter 节点自动伸缩](docs/demo09-karpenter.md)

### 4. 存储、身份与可观测性

- [Demo11 — 使用 CSI 部署有状态应用（EBS、EFS、S3）](docs/demo11-csi-storage.md)
- [Demo12 — 身份与访问控制](docs/demo12-iam-access.md)（Secrets Store CSI 部分选做）
- [Demo14 — CloudWatch Observability 原生可观测性](docs/demo14-cloudwatch.md)

## 进阶

### 5. 可靠性、入口与计算形态

- [Demo05 — Velero 备份恢复](docs/demo05-velero-backup.md)
- [Demo10 — Cluster Autoscaler 节点自动伸缩](docs/demo10-cluster-autoscaler.md)
- [Demo15 — 管理计算节点与 Fargate](docs/demo15-nodes-fargate.md)

### 6. 安全、网络与可观测性扩展

- [Demo16 — 使用 NetworkPolicy 限制 Pod 流量](docs/demo16-network-policy.md)
- [Demo17 — Pod Security Standards 工作负载安全](docs/demo17-pod-security.md)
- [Demo13 — Prometheus 与 Grafana 集群监控](docs/demo13-prometheus-grafana.md)
- [Demo18 — ADOT 与 OpenTelemetry 可观测性](docs/demo18-adot-otel.md)

---

## 使用方式

1. 在此目录下打开 Claude Code，`CLAUDE.md` 自动加载全球区配置
2. 将对应 [`docs/`](docs/) 目录中的 Demo 文件内容粘贴到对话框，由 AI 自主执行
3. 每个 Demo 末尾均有**清理**步骤，实验结束后执行以避免持续计费

与中国区版本（`china-eks-quickstart/`）的差异：

| 项目 | 全球区 | 中国区 |
|------|--------|--------|
| IAM ARN | `arn:aws:` | `arn:aws-cn:` |
| ECR 域名 | `.amazonaws.com` | `.amazonaws.com.cn` |
| 工具安装 | 直接从 GitHub/官方源下载 | 从 S3 工具桶预置包 |
| 镜像仓库 | docker.io / quay.io / registry.k8s.io 均可访问 | 受限，需使用 public.ecr.aws 或私有 ECR |
| container-mirror | 不需要 | 仅 Demo06（ArgoCD）需要 |
| ELB 端口 | 80 / 443 可用 | 需备案，建议 8080 |

---

## 环境要求

| 工具 | 版本 |
|------|------|
| AWS CLI | 2.x |
| eksctl | latest |
| kubectl | v1.35 |
| Helm | 3.x |
| jq | latest |
| Docker | 24.x+ |

操作机建议使用 Amazon Linux 2023 EC2，绑定具备 EKS/EC2/IAM 操作权限的 IAM Role。

所需 IAM 权限参考 [eksctl minimum IAM policies](https://eksctl.io/usage/minimum-iam-policies/)。

## License

MIT - see the [LICENSE](LICENSE) file for details.

## 免责声明

- 本项目仅供学习与技术参考，不构成生产部署方案。
- 运行过程中会创建 AWS 资源并产生费用，请在实验结束后及时清理。
- 作者不对因使用本项目产生的任何费用或损失承担责任。
- 本项目与 Amazon Web Services 无官方关联，相关服务的可用性与定价以 AWS 官方文档为准。
- 生产环境使用前请根据实际需求进行安全评估与调整。