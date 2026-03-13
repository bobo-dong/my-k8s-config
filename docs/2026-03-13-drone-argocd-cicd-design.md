# Drone CI + Argo CD 完整 CI/CD 流程设计

**日期**: 2026-03-13
**项目**: my-demo-app
**目标**: 建立从代码提交到 Kubernetes 部署的完整 GitOps 流程

## 1. 架构概览

### 完整流程

```
开发者 Push 代码 → GitHub (my-demo-app)
    ↓
[跳过 Drone CI - 手动构建]
    ↓
手动构建 Docker 镜像
    ↓
推送到 Docker Hub (woobamboo/my-demo-app:版本号)
    ↓
Argo CD 监控 my-k8s-config 仓库（手动同步模式）
    ↓
用户在 Argo CD UI 手动点击 "Sync"
    ↓
Argo CD 应用 deployment.yaml 到 Kubernetes
    ↓
Kubernetes 执行滚动更新（2 个副本，零停机）
    ↓
用户验证：Argo CD UI + port-forward 访问应用
```

### 核心原则

- **GitOps**: 所有 K8s 配置存储在 Git (my-k8s-config)
- **单向数据流**: 代码 → 镜像 → Git 配置 → 集群
- **手动控制**: Argo CD 手动同步，用户决定部署时机
- **滚动更新**: 零停机部署，保持服务可用性

## 2. 组件配置

### 2.1 Argo CD Application

**名称**: my-demo-app
**命名空间**: argocd
**源仓库**: https://github.com/bobo-dong/my-k8s-config.git
**路径**: `.` (根目录)
**目标集群**: in-cluster (本地 K8s)
**目标命名空间**: default

**同步策略**:
- 手动同步（不配置 automated）
- 允许修剪（prune: true）
- 禁用自愈（selfHeal: false）

**YAML 配置**:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-demo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/bobo-dong/my-k8s-config.git
    targetRevision: main
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    syncOptions:
    - CreateNamespace=true
    - PruneLast=true
```

### 2.2 Kubernetes 部署配置

**Deployment** (my-k8s-config/deployment.yaml):
- 副本数: 2
- 镜像: woobamboo/my-demo-app:1.0.0
- 端口: 3000
- 健康检查: /health (liveness + readiness)
- 资源限制: CPU 100m-200m, Memory 64Mi-128Mi
- 更新策略: RollingUpdate (默认)

**Service** (my-k8s-config/service.yaml):
- 类型: ClusterIP
- 端口映射: 80 → 3000
- 选择器: app=my-demo-app

## 3. 端到端测试流程

### 步骤 0: 验证前置条件

**检查 Docker Hub 镜像**:
```bash
docker images | grep my-demo-app
# 应该看到: woobamboo/my-demo-app:1.0.0 和 latest
```

**检查 Argo CD 运行状态**:
```bash
kubectl get pods -n argocd
# 所有 Pod 应该是 Running
```

**访问 Argo CD UI**:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# 访问: https://localhost:8080
```

### 步骤 1: 创建 Argo CD Application

```bash
# 应用 Application 配置
kubectl apply -f argocd-application.yaml

# 验证创建成功
kubectl get application -n argocd my-demo-app
```

**在 Argo CD UI 中验证**:
- 应该看到 my-demo-app 应用
- 初始状态: OutOfSync (Git 中的配置与集群不一致)
- Health: Unknown 或 Missing (资源还未创建)

### 步骤 2: 首次同步部署

**在 Argo CD UI 中**:
1. 点击 my-demo-app 应用
2. 点击右上角 "SYNC" 按钮
3. 在弹出框中确认同步选项
4. 点击 "SYNCHRONIZE"

**观察同步过程**:
- 创建 Deployment: my-demo-app
- 创建 Service: my-demo-app
- Pod 启动: 2 个副本从 0/2 → 1/2 → 2/2 Running

**验证状态**:
- Sync Status: Synced (绿色勾号)
- Health Status: Healthy (绿色心形)
- Pods: 2/2 Running

### 步骤 3: 触发版本更新测试

**修改应用代码**:
```bash
cd ~/Documents/dodo/my-demo-app

# 修改 index.js 中的 message
# 例如: "Hello from my-demo-app v2!"

# 更新版本号
# 编辑 package.json: "version": "1.0.1"
```

**构建并推送新镜像**:
```bash
docker build -t woobamboo/my-demo-app:1.0.1 .
docker tag woobamboo/my-demo-app:1.0.1 woobamboo/my-demo-app:latest
docker push woobamboo/my-demo-app:1.0.1
docker push woobamboo/my-demo-app:latest
```

**更新 K8s 配置**:
```bash
cd ~/Documents/dodo/my-k8s-config

# 修改 deployment.yaml 中的镜像版本
sed -i '' 's/my-demo-app:1.0.0/my-demo-app:1.0.1/g' deployment.yaml

# 提交并推送
git add deployment.yaml
git commit -m "Update image to 1.0.1"
git push origin main
```

### 步骤 4: Argo CD 检测变化

**在 Argo CD UI 中**:
1. 刷新应用页面（或等待 3 分钟自动刷新）
2. 观察状态变化:
   - Sync Status: OutOfSync (黄色)
   - 点击 "APP DIFF" 查看差异
   - 应该显示镜像版本从 1.0.0 → 1.0.1

### 步骤 5: 手动同步新版本

**触发同步**:
1. 点击 "SYNC" 按钮
2. 确认同步

**观察滚动更新过程**:
```
初始状态: 2 个 Pod 运行 1.0.0 版本
    ↓
创建新 Pod (1.0.1): Pod-3 创建
    ↓
等待 Pod-3 Ready (健康检查通过)
    ↓
终止旧 Pod: Pod-1 终止
    ↓
创建另一个新 Pod (1.0.1): Pod-4 创建
    ↓
等待 Pod-4 Ready
    ↓
终止最后旧 Pod: Pod-2 终止
    ↓
最终状态: 2 个 Pod 运行 1.0.1 版本
```

**关键观察点**:
- 在整个过程中至少有 1 个 Pod 可用
- 新 Pod 必须通过健康检查才能接管流量
- 旧 Pod 在新 Pod Ready 后才终止

### 步骤 6: 验证部署成功

**Argo CD UI 验证**:
- Sync Status: Synced
- Health Status: Healthy
- Pods: 2/2 Running
- 镜像版本: woobamboo/my-demo-app:1.0.1

**应用功能验证**:
```bash
# 转发服务到本地
kubectl port-forward svc/my-demo-app -n default 3000:80

# 在另一个终端测试
curl http://localhost:3000/
# 应该看到新的 message: "Hello from my-demo-app v2!"

curl http://localhost:3000/health
# 应该返回: {"status":"healthy"}
```

**检查 Pod 日志**:
```bash
kubectl get pods -n default -l app=my-demo-app
kubectl logs -n default <pod-name>
# 应该看到: "Server running on port 3000"
```

## 4. 故障排除

### 问题 1: Application 状态 Unknown

**症状**: Argo CD Application 显示 "Unknown" 状态

**排查**:
```bash
kubectl describe application my-demo-app -n argocd
```

**可能原因**:
- Git 仓库无法访问（检查 URL 是否正确）
- 仓库是私有的（需要配置 SSH key 或 token）
- 路径配置错误（检查 spec.source.path）

**解决方案**:
- 确认仓库 URL: https://github.com/bobo-dong/my-k8s-config.git
- 确认仓库是公开的
- 在 Argo CD UI 中点击 "REFRESH" 强制重新获取

### 问题 2: Pod ImagePullBackOff

**症状**: Pod 无法启动，状态为 ImagePullBackOff

**排查**:
```bash
kubectl describe pod <pod-name> -n default
# 查看 Events 部分的错误信息
```

**可能原因**:
- 镜像不存在于 Docker Hub
- 镜像名称或标签错误
- 镜像是私有的但未配置 imagePullSecrets

**解决方案**:
```bash
# 验证镜像存在
docker pull woobamboo/my-demo-app:1.0.0

# 检查 deployment.yaml 中的镜像名称
# 应该是: woobamboo/my-demo-app:1.0.0
```

### 问题 3: Pod 不健康 (Unhealthy)

**症状**: Pod Running 但 Argo CD 显示 Unhealthy

**排查**:
```bash
kubectl get pods -n default -l app=my-demo-app
kubectl logs <pod-name> -n default
kubectl describe pod <pod-name> -n default
```

**可能原因**:
- 健康检查端点 /health 返回错误
- 应用启动失败
- 端口配置错误

**解决方案**:
```bash
# 检查应用日志
kubectl logs <pod-name> -n default

# 手动测试健康检查
kubectl port-forward <pod-name> -n default 3001:3000
curl http://localhost:3001/health
```

### 问题 4: 滚动更新卡住

**症状**: 新 Pod 一直处于 Pending 或 ContainerCreating

**排查**:
```bash
kubectl get pods -n default -l app=my-demo-app
kubectl describe pod <new-pod-name> -n default
```

**可能原因**:
- 资源不足（CPU/Memory）
- 镜像拉取超时
- 健康检查配置过于严格

**解决方案**:
```bash
# 检查节点资源
kubectl top nodes
kubectl describe nodes

# 调整健康检查延迟
# 修改 deployment.yaml:
# initialDelaySeconds: 10 → 30
```

### 问题 5: Argo CD 不显示 OutOfSync

**症状**: Git 已更新但 Argo CD 仍显示 Synced

**排查**:
- 检查 Argo CD 的刷新时间间隔（默认 3 分钟）
- 检查 Git commit 是否真的推送成功

**解决方案**:
```bash
# 手动刷新
# 在 Argo CD UI 中点击 "REFRESH" 按钮

# 或使用 CLI
argocd app get my-demo-app --refresh
```

## 5. 访问方式

### Argo CD UI

**通过 port-forward**:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# 访问: https://localhost:8080
```

**获取初始密码**:
```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

### 应用访问

**通过 port-forward**:
```bash
kubectl port-forward svc/my-demo-app -n default 3000:80
# 访问: http://localhost:3000
```

**端点**:
- `/` - 主页，返回 JSON (message, version, timestamp)
- `/health` - 健康检查，返回 {"status":"healthy"}

## 6. 后续优化建议

### 6.1 自动同步（可选）

如果希望 Argo CD 自动同步更新：

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
  - CreateNamespace=true
```

### 6.2 添加 Ingress

暴露服务到集群外部：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-demo-app
spec:
  rules:
  - host: my-demo-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-demo-app
            port:
              number: 80
```

### 6.3 修复 Drone CI

未来解决 webhook 问题后，可以实现完全自动化的 CI/CD。

### 6.4 多环境支持

添加 staging 和 production 环境：

```
my-k8s-config/
├── base/
│   ├── deployment.yaml
│   └── service.yaml
├── overlays/
│   ├── staging/
│   │   └── kustomization.yaml
│   └── production/
│       └── kustomization.yaml
```

## 7. 成功标准

部署成功的标志：

✅ Argo CD Application 状态: Synced + Healthy
✅ Kubernetes Pods: 2/2 Running
✅ 访问 http://localhost:3000 返回正确内容
✅ 访问 http://localhost:3000/health 返回 healthy
✅ 滚动更新时服务不中断
✅ 新版本部署后能看到更新的 message

## 8. 学习目标达成

通过这个流程，你将理解：

- ✅ GitOps 原理：配置即代码，Git 作为唯一真实来源
- ✅ Argo CD 工作原理：监控 Git → 同步到集群
- ✅ Kubernetes 滚动更新：零停机部署策略
- ✅ 健康检查：如何确保 Pod 就绪后才接管流量
- ✅ CI/CD 流程：从代码到部署的完整链路

## 9. 下一步行动

1. 创建 Argo CD Application YAML
2. 应用到集群
3. 执行首次同步
4. 测试版本更新流程
5. 验证滚动更新和健康检查
6. 熟悉故障排除方法
