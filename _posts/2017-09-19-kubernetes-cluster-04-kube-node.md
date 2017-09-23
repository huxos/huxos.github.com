---
layout: post
title: kubernetes集群搭建（4. kube node）
description: kubernetes集群搭建
categories:
  - Kubernetes
---

本例Kube Node节点的kubelet通过VIP 10.22.108.250与APIServer通讯。kubelet第一次通过启动tls bootstrap认证后，由apiserver生成节点的证书。

#### 部署步骤


1.	网络组建部署

	参照上篇[kubernetes集群搭建（3. Kube master）](http://huxos.me/kubernetes/2017/09/19/kubernetes-cluster-03-kube-master.html) 配置好flanneld 以及CNI

2.	部署kubelet

	1.	生成kubectl获取认证的bootstrap-kubeconfig

		使用如下命令生成`bootstrap-kubeconfig.yaml`

		```
		kubectl config set-cluster kubernetes \
		    --certificate-authority=cert/ca.pem \
		    --embed-certs=true \
		    --server=http://10.22.108.250:6443 \
		    --kubeconfig=bootstrap-kubeconfig.yaml

		kubectl config set-credentials kubelet-bootstrap \
		    --token=${BOOTSTRAP} \
		    --kubeconfig=bootstrap-kubeconfig.yaml

		kubectl config set-context default \
		    --cluster=kubernetes \
		    --user=kubelet-bootstrap \
		    --kubeconfig=bootstrap-kubeconfig.yaml

		kubectl config use-context default --kubeconfig=bootstrap-kubeconfig.yaml
		```

		`--embed-certs=true` 选项可以让生成的证书.pem 文件内嵌到yaml文件中，简化配置文件。
        生成bootstrap-kubeconfig.yaml之后传输到node节点上，放到`/ete/kubernetes/` 目录中。

	2. 下载kubelet

		到node节点，使用下面的命令下载kubelet

		```
		wegt https://storage.googleapis.com/kubernetes-release/release/v1.7.3/bin/linux/amd64/kubelet -O /opt/bin/kubelet && \
        chmod +x /opt/bin/kubelet
		```
	3. 配置 kubelet 启动的systemd unit

		编辑`/lib/systemd/system/kubelet.service`:

		```
		[Service]
		ExecStartPre=/bin/mkdir -p /etc/kubernetes/manifests
		ExecStart=/opt/bin/kubelet \
		  --require-kubeconfig \
		  --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubeconfig.yaml \
		  --kubeconfig=/etc/kubernetes/worker-kubeconfig.yaml \
		  --cert-dir=/etc/kubernetes/ssl \
		  --container-runtime=docker \
		  --docker=unix:///var/run/docker.sock \
		  --network-plugin=cni \
		  --allow-privileged=true \
		  --pod-manifest-path=/etc/kubernetes/manifests \
		  --hostname-override=10.22.108.80 \
		  --cluster-dns=10.99.66.2 \
		  --cluster-domain=cluster.local \
		  --max-pods=32 \
		  --serialize-image-pulls=false \
		  --pod-infra-container-image=gcr.io/google_containers/pause-amd64:3.0
		Restart=always
		RestartSec=10

		[Install]
		WantedBy=multi-user.target
		```

		- `--hostname-override=10.22.108.80 ` 此处更改为node节点的IP

        - `--kubeconfig=/etc/kubernetes/worker-kubeconfig.yaml --cert-dir=/etc/kubernetes/ssl`
        kubelet认证成功之后将会生成/etc/kubernetes/worker-kubeconfig.yaml， 并且在/etc/kubernetes/ssl 目录下面生成证书。

        - 使用`systemctl enable  kubelet && systemctl start kubelet` 启动kubelet。

	4.	通过 kubelet 的证书请求

		```
		# kubectl get csr
		NAME                                                   AGE       REQUESTOR           CONDITION
		node-csr-K9fSKHorWHsmTl4X7iApDB5DPAEEidAnmt93ia8Aydk   2m       kubelet-bootstrap   Pending
		# kubectl certificate approve node-csr-K9fSKHorWHsmTl4X7iApDB5DPAEEidAnmt93ia8Aydk
		# kubectl get node
		```

3.	部署kube-proxy

	1.	生成kube-proxy使用的kubeconfig

		上篇[kubernetes集群搭建（3. Kube master）](http://huxos.me/kubernetes/2017/09/19/kubernetes-cluster-03-kube-master.html) 生成了kube-proxy所使用的证书文件，现在利用证书生成proxy-kubeconfig.yaml。

		```
		kubectl config set-cluster kubernetes \
		    --certificate-authority=cert/ca.pem \
		    --embed-certs=true \
		    --server=https://10.22.108.250:6443
		    --kubeconfig=proxy-kubeconfig.yaml

		kubectl config set-credentials proxy \
		    --client-certificate=cert/proxy.pem \
		    --client-key=cert/proxy-key.pem \
		    --embed-certs=true
		    --kubeconfig=proxy-kubeconfig.yaml

		kubectl config set-context default \
		    --cluster=kubernetes \
		    --user=porxy
		    --kubeconfig=proxy-kubeconfig.yaml

		kubectl config use-context default --kubeconfig=proxy-kubeconfig.yaml
		```
		将proxy-kubeconfig.yaml 拷贝到node节点的`/etc/kubernetes/`目录下。

	2.	配置kube-proxy启动的static pod

		编辑`/etc/kubernetes/manifests/kube-proxy.yaml`:

		```
		apiVersion: v1
		kind: Pod
		metadata:
		  name: kube-proxy
		  namespace: kube-system
		spec:
		  hostNetwork: true
		  containers:
		  - name: kube-proxy
		    image: quay.io/coreos/hyperkube:v1.7.3_coreos.0
		    command:
		    - /hyperkube
		    - proxy
		    - --hostname-override=10.22.108.80
		    - --cluster-cidr=10.99.66.0/23
		    - --kubeconfig=/etc/kubernetes/proxy-kubeconfig.yaml
		    securityContext:
		      privileged: true
		    volumeMounts:
		    - mountPath: /etc/ssl/certs
		      name: ssl-certs-host
		      readOnly: true
		    - mountPath: /etc/kubernetes/proxy-kubeconfig.yaml
		      name: proxy-kubeconfig
		  volumes:
		  - hostPath:
		      path: /usr/share/ca-certificates
		    name: ssl-certs-host
		  - hostPath:
		      path: /etc/kubernetes/proxy-kubeconfig.yaml
		    name: proxy-kubeconfig
		```

		`--hostname-override=10.22.108.80` 此处需要修改成节点的IP，由于POD需要操作iptables 所以需要：`privileged: true` 。

至此kube node节点就部署完成了。
