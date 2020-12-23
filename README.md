# prometheus-helm

 helm 安装 prometheus

## 准备

在使用 helm 安装是遇到 helm2 版本问题,故而先进行 helm3升级.

```bash
VERSION=3.4.2
wget -O https://get.helm.sh/helm-v${VERSION}-linux-amd64.tar.gz
```

二进制文件下载完成后解压,放在可执行目录下.

## 升级

### 安装升级插件

```bash
helm3 plugin install https://github.com/helm/helm-2to3
helm3 plugin list
```

### 迁移 Helm V2 配置

```bash
helm3 2to3 move config
helm3 repo list
```

### helm2 应用迁移

获取现有 helm2 应用

```bash
helm list
```

升级应用

```bash
# 校验
helm3 2to3 convert <helm2-app-name> --dry-run
# 校验成功后执行
helm3 2to3 convert <helm2-app-name> --tiller-out-cluster
```

### 清理

```bash
 helm3 2to3 cleanup
```

## 自定义 value 安装

获取 value 文件

```bash
helm3 inspect values prometheus-community/kube-prometheus-stack > myvalue.yaml
```

根据需要修改后, 安装

```bash
kubectl delete crd prometheuses.monitoring.coreos.com
kubectl delete crd prometheusrules.monitoring.coreos.com
kubectl delete crd servicemonitors.monitoring.coreos.com
kubectl delete crd podmonitors.monitoring.coreos.com
kubectl delete crd alertmanagers.monitoring.coreos.com
kubectl delete crd thanosrulers.monitoring.coreos.com
kubectl delete crd probes.monitoring.coreos.com
kubectl create ns monitor
helm3 delete prometheus -n monitor
# 要修改: sc 值:test-storage-class
helm3 install prometheus --namespace=monitor prometheus-community/kube-prometheus-stack -f myvalue.yaml

helm3 upgrade prometheus -n monitor prometheus-community/kube-prometheus-stack -f myvalue.yaml \
  --set alertmanager.service.type=LoadBalancer \
  --set prometheus.service.type=LoadBalancer \
  --set prometheusSpec.retention=120h \
  --set prometheusSpec.replicas=2
```

LoadBalancer方式暴露服务

```bash
kubectl patch svc prometheus-grafana -n monitor -p '{"spec": {"type": "ClusterIP"}}'
kubectl patch svc prometheus-grafana -n monitor -p '{"spec": {"type": "LoadBalancer"}}'
```

## 遇坑说明

- prometheus-kube-stack: service "prometheus-operator-kube-p-operator" not found

找到校验的 webhook 删除即可

```bash
kubectl get validatingwebhookconfigurations.admissionregistration.k8s.io
```

- unknown field "probeNamespaceSelector" in com.coreos.monitoring.v1.Prometheus.spec

之前遗留的 crd 资源可能由问题,清理后再安装

```bash
kubectl delete crd prometheuses.monitoring.coreos.com
kubectl delete crd prometheusrules.monitoring.coreos.com
kubectl delete crd servicemonitors.monitoring.coreos.com
kubectl delete crd podmonitors.monitoring.coreos.com
kubectl delete crd alertmanagers.monitoring.coreos.com
kubectl delete crd thanosrulers.monitoring.coreos.com
kubectl delete crd probes.monitoring.coreos.com
```

```bash
  --set prometheusOperator.admissionWebhooks.enabled=false \
  --set prometheusOperator.admissionWebhooks.patch.enabled=false \
  --set prometheusOperator.tlsProxy.enabled=false
```
