# My K8s Config

Kubernetes 配置仓库，用于 GitOps 部署。

## 包含的资源

- `deployment.yaml`: 应用部署配置
- `service.yaml`: 服务配置

## Argo CD

Argo CD 会监听这个仓库，当镜像版本更新时自动部署到集群。
