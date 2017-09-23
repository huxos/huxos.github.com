---
layout: post
title: kubernetes集群搭建（7. ingress）
description: kubernetes集群搭建
categories:
  - Kubernetes
---

> An [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/#what-is-ingress) is a collection of rules that allow inbound connections to reach the cluster services.

Ingress的引入主要解决创建入口站点规则的问题，主要作用于7层入口(http)。
可以通过K8s的Ingress对象定义类似于nginx中的vhost、localtion、upstream等。
Nginx官方也有Ingress的实现[nginxinc/kubernetes-ingress](https://github.com/nginxinc/kubernetes-ingress/releases)来对接k8s。

> [traefik](https://github.com/containous/traefik) Træfik (pronounced like traffic) is a modern HTTP reverse proxy and load balancer made to deploy microservices with ease. It supports several backends (Docker, Swarm mode, Kubernetes, Marathon, Consul, Etcd, Rancher, Amazon ECS, and a lot more) to manage its configuration automatically and dynamically.

考虑到Traefik部署较为方便，使用traefik提供Ingress服务。

#### 部署步骤

1.	定义traefik需要的RBAC规则

	```
	apiVersion: v1
	kind: ServiceAccount
	metadata:
  	  name: traefik-ingress-controller
  	  namespace: kube-system
	---
	kind: ClusterRole
	apiVersion: rbac.authorization.k8s.io/v1beta1
	metadata:
	  name: traefik-ingress-controller
	rules:
	  - apiGroups:
	      - ""
	    resources:
	      - pods
	      - services
	      - endpoints
	      - secrets
	    verbs:
	      - get
	      - list
	      - watch
	  - apiGroups:
	      - extensions
	    resources:
	      - ingresses
	    verbs:
	      - get
	      - list
	      - watch
	---
	kind: ClusterRoleBinding
	apiVersion: rbac.authorization.k8s.io/v1beta1
	metadata:
	  name: traefik-ingress-controller
	roleRef:
	  apiGroup: rbac.authorization.k8s.io
	  kind: ClusterRole
	  name: traefik-ingress-controller
	subjects:
	- kind: ServiceAccount
	  name: traefik-ingress-controller
	  namespace: kube-system
	```

2. 定义ingress编排的daemonset模版

	```
	apiVersion: extensions/v1beta1
	kind: DaemonSet
	metadata:
	  labels:
	    k8s-app: traefik-ingress-lb
	  name: traefik-ingress-controller
	  namespace: kube-system
	spec:
	  selector:
	    matchLabels:
	      k8s-app: traefik-ingress-lb
	  template:
	    metadata:
	      labels:
	        k8s-app: traefik-ingress-lb
	        name: traefik-ingress-lb
	    spec:
	      containers:
	      - args:
	        - --web
	        - --web.address=:8580
	        - --kubernetes
	        - --web.metrics
	        - --web.metrics.prometheus
	        image: traefik
	        imagePullPolicy: IfNotPresent
	        name: traefik-ingress-lb
	        ports:
	        - containerPort: 80
	          hostPort: 80
	          protocol: TCP
	        - containerPort: 8580
	          hostPort: 8580
	          protocol: TCP
	        resources:
	          requests:
	            cpu: "2"
	            memory: 4G
	      dnsPolicy: ClusterFirst
	      hostNetwork: true
	      nodeSelector:
	        role: ingress
	      restartPolicy: Always
	      schedulerName: default-scheduler
	      serviceAccount: traefik-ingress-controller
	      serviceAccountName: traefik-ingress-controller
	      terminationGracePeriodSeconds: 60
	```

	采用Host Network的方式，部署traekfik  
    通过`kubectl label node <NODE> role=ingress` 为节点打上相应的标签。

3.	创建ingress规则

	```
	apiVersion: extensions/v1beta1
	kind: Ingress
	metadata:
	  name: metadata
	  namespace: default
	spec:
	  rules:
	  - host: metadata.example.com
	    http:
	      paths:
	      - backend:
	          serviceName: metadata-server
	          servicePort: 80
	```

    这样其实相当于定义了一个http的站点，域名metadata.example.com 指向了default的metadata-server这个服务。

    访问相关节点的8580端口就能看到metadata.example.com站点对应的信息了。
