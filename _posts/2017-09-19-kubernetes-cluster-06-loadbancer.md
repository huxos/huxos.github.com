---
layout: post
title: kubernetes集群搭建（6. loadbancer）
description: kubernetes集群搭建
categories:
  - Kubernetes
---

> kubernetes在定义Service的时候提供了LoadBalancer类型，用于集群外部访问到kubernetes的服务。  
但是这里的LoadBalancer主要对接用云服务商提供的负载均衡服务，在物理机部署的环境就不是很适用了。  
结合kubernetes本身的Service就提供了负载均衡的功能，想到一个巧妙的方法: 把Service的Cluster IP做成可以在集群外部可以路由。  
可以在交换机上面配置等价路由，将Cluster IP段路由到kube-proxy的节点就行了，这种方式最多支持8个节点做集群的负载均衡。  
考虑到kube-proxy的冗长的iptables规则，当service的数目多了之后性能上会存在问题，所以采用kube-router作为负载均衡节点。  
kube-router基于IPVS实现，转发效率上会更高，规则可读性也会更好。  
如果集群网络是采用overlay，那么这种方式的可能的瓶颈在与做负载均衡的节点对集群内外的流量解封包的过程，小规模使用起来应该是没有问题。

> 另外即将发布的kubernetes 1.8中kube-proxy原生就支持IPVS，到时可以少引入kube-router这依赖了。

[kube-router](https://github.com/cloudnativelabs/kube-router) A distributed load balancer, firewall and router designed for Kubernetes。


根据官网的介绍kube-router有三个部分组成

- 第一部分：`--run-service-proxy` 基于IPVS的负载均衡器（IPVS/LVS based service proxy）可用于替换掉kube-proxy，用作kubenetes中的service的负载均衡。

- 第二部分：`--run-router` 基于BGP实现的跨主机网络（Pod Networking）

- 第三部分：`--run-firewall` 网络访问策略控制器（Network Policy Controller），基于ipset和iptables实现。


我们仅使用其了service-proxy的功能。  
本例部署在10.22.108.100、10.22.108.101 这两个机器上。  
服务启动之后会使用ClusterIP配置LVS的virtual server，并且在新增虚拟网口上绑定ClusterIP，设置好ipvs的规则，通过LVS使用nat方式将流量转发到容器中。  
我们在上层交换机上指定两条等价路由，将ClusterIP的地址段指向这两个机器，并且做好存活检测。

#### 部署步骤

1. 到入kube-router使用的rbac规则


    为了简化操作我们让`kube-router`使用前篇[kubernetes集群搭建（4. kube node）](http://huxos.me/kubernetes/2017/09/19/kubernetes-cluster-04-kube-node.html)
    中创建的`proxy-kubeconfig.yaml`来访问apiserver。  
    需要创建好kube-router需要的RBAC规则授权给`system:kube-proxy`这个用户。

	```
	kind: ClusterRole
	apiVersion: rbac.authorization.k8s.io/v1beta1
	metadata:
	  name: kube-router
	  namespace: kube-system
	rules:
	  - apiGroups:
	    - ""
	    resources:
	      - namespaces
	      - pods
	      - services
	      - nodes
	      - endpoints
	    verbs:
	      - list
	      - get
	      - watch
	  - apiGroups:
	    - "networking.k8s.io"
	    resources:
	      - networkpolicies
	    verbs:
	      - list
	      - get
	      - watch
	  - apiGroups:
	    - extensions
	    resources:
	      - networkpolicies
	    verbs:
	      - get
	      - list
	      - watch
	---
	kind: ClusterRoleBinding
	apiVersion: rbac.authorization.k8s.io/v1beta1
	metadata:
	  name: kube-router
	roleRef:
	  apiGroup: rbac.authorization.k8s.io
	  kind: ClusterRole
	  name: kube-router
	subjects:
	- kind: User
	  name: system:kube-proxy
	  namespace: kube-system
	```

	将proxy-kubeconfig.yaml拷贝到两个机器的`/etc/kubernetes/`目录。

2. 创建kube-router的Daemonset

    我们创建kube-router的调度模版，将kube-router调度到label为`role=lb` 的机器上。  
    需要把`/etc/kubernetes/proxy-kubeconfig.yaml`挂载到容器中，kube-router会读取此文件的配置访问apiserver。  
    需要特权模式运行，并且把`/lib/modules`挂载到容器中。

	```
	apiVersion: extensions/v1beta1
	kind: DaemonSet
	metadata:
	  labels:
	    k8s-app: kube-router
	  name: kube-router
	  namespace: kube-system
	spec:
	  revisionHistoryLimit: 10
	  selector:
	    matchLabels:
	      k8s-app: kube-router
	  template:
	    metadata:
	      annotations:
	        scheduler.alpha.kubernetes.io/critical-pod: ""
	      labels:
	        k8s-app: kube-router
	    spec:
	      containers:
	      - args:
	        - --kubeconfig=/var/lib/kube-router/kubeconfig
	        - --run-service-proxy=true
	        - --run-router=false
	        - --run-firewall=false
	        env:
	        - name: NODE_NAME
	          valueFrom:
	            fieldRef:
	              apiVersion: v1
	              fieldPath: spec.nodeName
	        image: cloudnativelabs/kube-router:v0.0.12
	        imagePullPolicy: IfNotPresent
	        name: kube-router
	        securityContext:
	          privileged: true
	        volumeMounts:
	        - mountPath: /lib/modules
	          name: lib-modules
	          readOnly: true
	        - mountPath: /var/lib/kube-router/kubeconfig
	          name: kubeconfig
	      dnsPolicy: ClusterFirst
	      hostNetwork: true
	      nodeSelector:
	        role: lb
	      restartPolicy: Always
	      volumes:
	      - hostPath:
	          path: /lib/modules
	        name: lib-modules
	      - hostPath:
	          path: /etc/kubernetes/proxy-kubeconfig.yaml
	        name: kubeconfig
	```

    为两个机器打上相应的label，等待启动成功我们可以是用`ipvsadm -Ln`来查看规则是否生效。

3. 创建SNAT规则

    等待上面的容器启动之后发现不能正常工作，经查发现IPVS实现了DNAT，只对目的地址执行了转换，要使得POD正常回包，还需要再设置SNAT。

	```
	iptables -t nat -A POSTROUTING -m ipvs --vdir ORIGINAL --vmethod MASQ -m comment --comment "ipvs snat rule" -j MASQUERADE
	```

    这点有点像Full NAT模式，只不过SNAT由iptables完成的。

4. 配置交换机等价路由

    最后需要在交换机上面配置好等价路由将clusterIP段 10.99.66.0/23 路由到这两个机器上就行了。

至此基于IPVS的和ClusterIP的负载均衡就搭建完成了。
