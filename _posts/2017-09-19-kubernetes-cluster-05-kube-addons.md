---
layout: post
title: kubernetes集群搭建（5. kube-dns、dashboard）
description: kubernetes集群搭建
categories:
  - Kubernetes
---

> kube-dns、dashboard以deployment的方式部署到kubernetes，运行在kube-system这个namespace中。
只需要导入相应的模版就行，部署较为简单。


1. 下载

	```
	# wget https://github.com/kubernetes/kubernetes/releases/download/v1.7.4/kubernetes.tar.gz  -O - |tar -zxpvf -
	# cd kubernetes
	```


2. kube-dns


	```
	~/kubernetes # cd cluster/addons/dns
	~/k/c/a/dns # rm -rf *.yaml.sed
	~/k/c/a/dns # cat  transforms2sed.sed
	s/__PILLAR__DNS__SERVER__/10.99.66.2/g
	s/__PILLAR__DNS__DOMAIN__/cluster.local/g
	s/__MACHINE_GENERATED_WARNING__/Warning: This is a file generated from the base underscore template file: __SOURCE_FILENAME__/g
	~/k/c/a/dns # make
	sed -f transforms2sed.sed kubedns-controller.yaml.base  | sed s/__SOURCE_FILENAME__/kubedns-controller.yaml.base/g > kubedns-controller.yaml.sed
	sed -f transforms2sed.sed kubedns-svc.yaml.base  | sed s/__SOURCE_FILENAME__/kubedns-svc.yaml.base/g > kubedns-svc.yaml.sed
	```

	- 进入`cluster/addons/dns`目录
    - 删除`*.yaml.sed`
    - 编辑`transforms2sed.sed` 将里面的`$DNS_SERVER`、`$DNS_DOMAIN`替换成相应的地址和集群名字
    - 然后执行`make`指令，就生成了kube-dns的模版
	- 最后导入模版到kubernetes 集群

        ```
        kubectl create -f  kubedns-cm.yaml -f kubedns-sa.yaml  -f kubedns-controller.yaml.sed -f kubedns-svc.yaml.sed
        ```

3.	dashboard 插件部署

	首先创建server-account

	```
	apiVersion: v1
	kind: ServiceAccount
	metadata:
	  labels:
	    k8s-app: kubernetes-dashboard
	  name: kubernetes-dashboard
	  namespace: kube-system
	```

	创建ClusterRoleBonding，分配k8s内置的view角色给它，访问到kube-dashbroad的用户拥有只读的权限。

	```
	apiVersion: rbac.authorization.k8s.io/v1beta1
	kind: ClusterRoleBinding
	metadata:
	  labels:
	    k8s-app: kubernetes-dashboard
	  name: kubernetes-dashboard
	roleRef:
	  apiGroup: rbac.authorization.k8s.io
	  kind: ClusterRole
	  name: view
	subjects:
	- kind: ServiceAccount
	  name: kubernetes-dashboard
	  namespace: kube-system
	```

	进入`cluster/addons/dashboard` 编辑 `dashboard-controller.yaml` 加入`serviceAccount: kubernetes-dashboard`。

	```
	....
	        livenessProbe:
	          httpGet:
	            path: /
	            port: 9090
	          initialDelaySeconds: 30
	          timeoutSeconds: 30
	      serviceAccount: kubernetes-dashboard #添加sa
	      tolerations:
	      - key: "CriticalAddonsOnly"
	        operator: "Exists"
    ....
	```

	最后执行

    ```
    kubectl create -f dashboard-controller.yaml dashboard-service.yaml
    ```

至此kube-dns、dashboard就部署完成了。
