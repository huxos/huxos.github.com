---
layout: post
title: kubernetes集群搭建（8. prometheus）
description: kubernetes集群搭建
categories:
  - Kubernetes
  - Prometheus
---


>[Prometheus](https://github.com/prometheus/prometheus) 的设计非常适合k8s集群的监控。  
大多数k8s的组件都提供prometheus格式的监控接口，只需要配置好kube-api作为promethues的sd就能非常容易的监控整个k8s集群，无需引入额外的依赖。  
prometheus提供了众多语言的库，可以非常容易的嵌入到业务代码中做业务监控。



可以采用CoreOS所提供的 [prometheus-operator](https://github.com/coreos/prometheus-operator)来部署prometheus，
但是使用过程中发现其有些较不灵活的地方。比如：只能使用promethues-operator预定义的参数启动prometheus。   
现将prometheus-operator生成的规则导出来做成configmap，独立部署prometheus，同时加入相关的规则使得其能够自动根据service的annotation进行发现和监控。


#### 部署步骤

使用statefulset的方式部署prometheus，监控数据存储在ceph的rbd里面。   
考虑到1.7版本的prometheus在数据轮换的时候产生大量的IO开销，所以部署了prometheus2.0 beta。


1. 一些准备工作

	首先创建monitoring这个namespaces

	```
	# kubectl create ns monitoring
	```

	参照前面的[kubernetes集群搭建（2. Ceph）](http://huxos.me/kubernetes/ceph/2017/09/19/kubernetes-cluster-02-ceph.html)，
    在monitoring这个namespace中要使用RBD先要作为secret导入ceph的key。

	```
	kubectl create secret generic ceph-secret-kube --type="kubernetes.io/rbd" \
        --from-literal=key=`ceph --cluster ceph auth get-key client.kube` --namespace=monitoring
	```

	由于etcd部署了ssl，prometheus监控etcd需要先导入etcd的证书到k8s中用secret保存

	```
	kubectl create secret generic etcd-client-ssl --from-file=ca.pem --from-file=client-key.pem --from-file=client.pem
	```

2. 部署prometheus

	部署prometheus的编排模版已经将其上传到了github上，首先将其clone下来。

	```
    git clone https://github.com/huxos/prometheus-kubernetes.git
	```

	部署prometheus:

	```
	kubectl create -f prometheus-rbac.yaml
	kubectl create -f prometheus-k8s-cm.yaml
	kubectl create -f prometheus-k8s-rules.yaml
	kubectl create -f prometheus-statefulset.yaml
	kubectl create -f prometheus-svc.yaml
	```

	部署alertmanager（alertmanager主要用来做prometheus的监控告警）：

	```
	kubectl create -f alertmanager-cm.yaml
	kubectl create -f alertmanager-statefulset.yaml
	kubectl create -f alertmanager-svc.yaml
	```

	部署kube-state-metric（kube-state-metric用来获取k8s集群的关联信息）:

	```
	kubectl create -f kube-state-metric-rbac.yaml
	kubectl create -f kube-state-metric-deploy.yaml
	kubectl create -f kube-state-metric-svc.yaml
	```

	部署node-exporter（如果需要宿主机的监控，需要部署node-exporter）：

	```
	kubectl create -f node-exporter-ds.yaml
	kubectl create -f node-exporter-svc.yaml
	```

	部署grafana（grafana用来做监控的绘图展示）：

	```
	kubectl create -f grafana-credentails.secret.yaml
	kubectl create -f grafana-deploy.yaml
	kubectl create -f grafana-svc.yaml
	```

	创建相关enpoints用于kubelet、kube-conrtroller-manger、kube-scheduler、etcd等的监控：

	```
	kubectl create -f prometheus-discovery-service.yaml

	```

	最后在grafan的页面（用户名：admin、密码：admin）mport文件夹dashborad里面的模版，就行了。

#### 监控集群内的服务

以ceph的监控为例，看看prometheus如何监控集群内的service。

首先部署好ceph-exporter、访问9128端口确保能正常获取数据、创建service让prometheus自动发现并采集ceph-exporter的监控信息：

```
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/scrape: "true"
   labels:
    k8s-app: ceph-exporter
    run: ceph-exporter
  name: ceph-exporter
  namespace: monitoring
spec:
  clusterIP: None
  ports:
  - name: metrics
    port: 9128
    protocol: TCP
    targetPort: 9128
  selector:
    run: ceph-exporter
  type: ClusterIP
```

在service的 annotations 中加入`prometheus.io/scrape: "true"` 就能使prometheus自动发现并监控。
此外还可以通过`prometheus.io/schema`、`prometheus.io/path`、`prometheus.io/port`等设置自动发现规则。
