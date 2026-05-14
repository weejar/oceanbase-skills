# ob-operator Kubernetes 部署

> 适用版本：ob-operator V2.x / OceanBase V4.2+ / Kubernetes V1.20+

---

## 概述

ob-operator 是 OceanBase 的 Kubernetes Operator，通过 CRD（Custom Resource Definition）实现 OceanBase 在 Kubernetes 上的自动化部署和运维。

| 功能 | 说明 |
|------|------|
| 自动部署 | 一键部署 OB 集群到 K8s |
| 弹性扩缩容 | 修改副本数自动扩缩容 |
| 滚动升级 | 声明式升级，自动滚动 |
| 故障自愈 | Pod 异常自动恢复 |
| 监控集成 | Prometheus + Grafana |
| 备份恢复 | S3/OSS 存储集成 |

---

## 1. 架构

```
Kubernetes Cluster
  ├── ob-operator (Deployment)
  │     └── 监听 OceanBaseCluster CRD
  ├── OceanBaseCluster CR
  │     ├── Zone1
  │     │     ├── OBServer Pod (1..N)
  │     │     └── OBProxy Pod (1..N)
  │     ├── Zone2
  │     │     └── ...
  │     └── Zone3
  │           └── ...
  └── OBServer PVC (每个 Pod 对应一个 PVC)
```

---

## 2. 安装 ob-operator

### 2.1 前置条件

```bash
# 检查 Kubernetes 版本（≥1.20）
kubectl version

# 检查 StorageClass
kubectl get sc

# 确保节点资源充足
kubectl top nodes
```

### 2.2 安装 Operator

```bash
# 方式1：Helm 安装（推荐）
helm repo add oceanbase https://oceanbase.github.io/ob-operator/
helm install ob-operator oceanbase/ob-operator \
  --namespace oceanbase-system \
  --create-namespace

# 方式2：YAML 安装
kubectl apply -f https://raw.githubusercontent.com/oceanbase/ob-operator/main/deploy/operator.yaml

# 验证
kubectl get pods -n oceanbase-system
# NAME                             READY   STATUS    RESTARTS   AGE
# ob-operator-xxxxxxxxxx-xxxxx     1/1     Running   0          2m
```

---

## 3. 部署 OceanBase 集群

### 3.1 准备 StorageClass

```yaml
# oceanbase-sc.yaml（示例，实际使用集群的 StorageClass）
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: oceanbase-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

### 3.2 创建 OceanBaseCluster

```yaml
# obcluster.yaml
apiVersion: oceanbase.oceanbase.com/v1alpha1
kind: OceanBaseCluster
metadata:
  name: ob-cluster
  namespace: oceanbase
spec:
  observer:
    image: oceanbase/oceanbase-ce:4.3.0
    replicas: 3
    resource:
      cpu: 8
      memory: 16Gi
    storage:
      dataStorage:
        storageClass: oceanbase-storage
        size: 200Gi
      logStorage:
        storageClass: oceanbase-storage
        size: 50Gi
      redoLogStorage:
        storageClass: oceanbase-storage
        size: 50Gi
    parameters:
      - name: memory_limit
        value: "12G"
      - name: system_memory
        value: "4G"
      - name: datafile_size
        value: "150G"
      - name: cpu_count
        value: "8"
  topology:
    - zone: zone1
      replicas: 1
    - zone: zone2
      replicas: 1
    - zone: zone3
      replicas: 1
  user:
    rootPassword: Root@Pass2024
```

```bash
# 部署
kubectl apply -f obcluster.yaml

# 查看 Pod 启动
kubectl get pods -n oceanbase -l app=oceanbase
kubectl logs -n oceanbase ob-cluster-zone1-0 -c observer -f
```

### 3.3 部署 OBProxy

```yaml
# obproxy.yaml
apiVersion: oceanbase.oceanbase.com/v1alpha1
kind: OBTenantProxy
metadata:
  name: ob-proxy
  namespace: oceanbase
spec:
  image: oceanbase/obproxy-ce:4.3.0
  replicas: 2
  clusterName: ob-cluster
  parameters:
    - name: obcluster
      value: "ob-cluster"
    - name: rs_list
      value: "ob-cluster-zone1-0.ob-cluster:2881;ob-cluster-zone2-0.ob-cluster:2881;ob-cluster-zone3-0.ob-cluster:2881"
```

```bash
kubectl apply -f obproxy.yaml
```

---

## 4. 创建租户

```yaml
# obtenant.yaml
apiVersion: oceanbase.oceanbase.com/v1alpha1
kind: OBTenant
metadata:
  name: business-tenant
  namespace: oceanbase
spec:
  clusterName: ob-cluster
  tenantName: business
  unitConfig:
    cpu: 4
    memory: 8Gi
  poolList:
    - zone: zone1
      replica: 1
    - zone: zone2
      replica: 1
    - zone: zone3
      replica: 1
  charset: utf8mb4
  compatibilityMode: MySQL
  rootPassword: BizRoot@2024
```

```bash
kubectl apply -f obtenant.yaml

# 查看 Tenant 状态
kubectl get obtenant -n oceanbase
```

---

## 5. 运维操作

### 5.1 扩容

```bash
# 修改 OBServer 副本数
kubectl edit oceanbasecluster ob-cluster -n oceanbase
# 修改 spec.observer.replicas: 3 → 5

# 修改租户资源
kubectl edit obtenant business-tenant -n oceanbase
# 修改 spec.unitConfig.cpu: 4 → 8
```

### 5.2 升级

```bash
# 修改镜像版本
kubectl edit oceanbasecluster ob-cluster -n oceanbase
# 修改 spec.observer.image: oceanbase/oceanbase-ce:4.3.1

# 自动触发滚动升级
kubectl get pods -n oceanbase -w
```

### 5.3 备份

```yaml
# obbackup.yaml
apiVersion: oceanbase.oceanbase.com/v1alpha1
kind: OBBackup
metadata:
  name: daily-backup
  namespace: oceanbase
spec:
  clusterName: ob-cluster
  tenantName: business
  destType: OSS
  ossAccessKey: ******
  ossSecretKey: ******
  ossEndpoint: oss-cn-hangzhou.aliyuncs.com
  ossBucket: ob-backup
  schedule: "0 2 * * *"  # 每天2点
```

---

## 6. 监控

```yaml
# ob-monitor.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ob-prometheus-config
data:
  prometheus.yml: |
    scrape_configs:
      - job_name: 'oceanbase'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app]
            regex: oceanbase
            action: keep
```

---

## 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| Pod Pending | PVC 绑定失败 | 检查 StorageClass 和节点 |
| 启动超时 | 资源不足 | 增加 CPU/内存 |
| Zone 不均衡 | 节点分布不均 | 检查 topology 配置 |
| 升级卡住 | 副本同步中 | 等待 Paxos 同步完成 |

---

## 参考文档

- ob-operator GitHub：https://github.com/oceanbase/ob-operator
- Helm Chart：https://artifacthub.io/packages/helm/oceanbase/ob-operator
- CRD 参考：https://github.com/oceanbase/ob-operator/tree/main/doc
