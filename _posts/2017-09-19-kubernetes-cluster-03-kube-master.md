---
layout: post
title: kubernetes集群搭建（3. Kube master）
description: kubernetes集群搭建
categories:
  - Kubernetes
---

本例中部署了kubernetes部署了Master三个节点，可以使用部署Keepalived做Apiserver的高可用。  
其中kubelet是采用systemd启动在物理机上面的，其他组件通过kubernetes的manifests启动在容器里面。  
三个Kubernetes Master的IP地址为：10.22.108.20、10.22.108.79、10.22.108.92，假定使用10.22.108.250作为Apiserver的VIP。

#### k8s集群配置

- kube-apiserver: 10.22.108.20,10.22.108.79,10.22.108.92, VIP: 10.22.108.250
- service-cidr: 10.99.66.0/23
- cluster-domain: cluster.local
- cluster-cidr: 10.20.0.0/16
- apiserver-service-ip: 10.99.66.1
- kubedns-service-ip: 10.99.66.2


#### 部署过程

1.  生成证书

	采用CloudFlare的PKI工具集 [cfssl](https://github.com/cloudflare/cfssl)来制作证书。

	1. 安装cfssl

        ```
        export GOPATH=${GOPATH:-$HOME/go}
        go get -u github.com/cloudflare/cfssl/cmd/...
        export PATH=$GOPATH/bin:$PATH
        ```

	2. 生成ca

		编辑ca-config.json：

		```
		{
		    "signing": {
		        "default": {
		            "expiry": "87600h"
		        },
		        "profiles": {
		            "server": {
		                "expiry": "87600h",
		                "usages": [
		                    "signing",
		                    "key encipherment",
		                    "server auth"
		                ]
		            },
		            "client": {
		                "expiry": "87600h",
		                "usages": [
		                    "signing",
		                    "key encipherment",
		                    "client auth"
		                ]
		            },
		            "peer": {
		                "expiry": "87600h",
		                "usages": [
		                    "signing",
		                    "key encipherment",
		                    "server auth",
		                    "client auth"
		                ]
		            }
		        }
		    }
		}
		```

		ca-csr.json

		```
		{
		    "CN": "kubernetes",
		    "key": {
		        "algo": "rsa",
		        "size": 2048
		    },
		    "names": [
		        {
		            "O": "KUBERNETES"
		        }
		    ]
		}
		```

		使用命令`cfssl gencert -initca ca-csr.json | cfssljson -bare ca -` 生成ca证书。

	3. 生成etcd的证书

        生成etcd-server证书，编辑server.json：

		```
		{
		    "CN": "server",
		    "hosts": [
		        "etcd",
		        "127.0.0.1",
		        "10.22.108.20",
		        "10.22.108.79",
		        "10.22.108.92"
		    ],
		    "key": {
		        "algo": "rsa",
		        "size": 2048
		    },
		    "names": [
		        {
		            "O": "ETCD"
		        }
		    ]
		}
		```

		使用如下命令生成etcd server证书:

        ```
        cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server server.json | cfssljson -bare server
        ```

        为每一个节点生成etcd peer证书，给etc集群内部通讯使用，执行如下脚本生成证书：

		```bash
		index=0
		for SERVER in "10.22.108.20" "10.22.108.79" "10.22.108.92"
		do
		cat <<EOF | cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer - | cfssljson -bare server${index}
		{
		    "CN": "infra${index}",
		    "hosts": [
		        "etcd",
		        "$SERVER",
		        "127.0.0.1"
		    ],
		    "key": {
		        "algo": "rsa",
		        "size": 2048
		    },
		    "names": [
		        {
		            "O": "ETCD"
		        }
		    ]
		}
		EOF
		index=`expr $index + 1`
		done
		```

		生成etcd client证书，编辑client.json：

		```
		{
		    "CN": "client",
		    "hosts": [
		        ""
		    ],
		    "key": {
		        "algo": "rsa",
		        "size": 2048
		    },
		    "names": [
		        {
		            "O": "ETCD"
		        }
		    ]
		}
		```

		使用如下命令生成etcd client证书，kube-apiserver访问etcd时使用。

        ```
        cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client client.json | cfssljson -bare client
        ```

	4. 生成kuberbetes相关的证书

		1. 生成api-server的证书

			编辑kube-apiserver.json:

			```
			{
			    "CN": "kube-apiserver",
			    "hosts": [
			        "kubernetes",
			        "kubernetes.default",
			        "kubernetes.default.svc",
			        "kubernetes.default.svc.cluster.local",
			        "127.0.0.1",
			        "10.101.66.1",
			        "10.22.108.20",
			        "10.22.108.79",
			        "10.22.108.92",
			        "10.22.108.250"
			    ],
			    "key": {
			        "algo": "rsa",
			        "size": 2048
			    },
			    "names": [
			        {
			            "O": "system:kube-apiserver"
			        }
			    ]
			}
			```

			使用下面的命令生成证书：

            ```
            cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server kube-apiserver.json | \
                cfssljson -bare apiserver
            ```

			\#`NOTE` 配置中的`hosts`要包含访问apiserver用得到的所有的域名和IP。

        2. 生成kube-proxy的证书

            编辑kube-proxy.json:

            ```
            {
                "CN": "system:kube-proxy",
                "hosts": [
                ],
                "key": {
                    "algo": "rsa",
                    "size": 2048
                },
                "names": [
                    {
                        "O": "KUBERNETES"
                    }
                ]
            }
            ```

            执行命令：

            ```
            cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client kube-proxy.json | cfssljson -bare proxy
            ```

        3. 生成kube-master证书

            编辑kube-master.json:

			```
			{
			    "CN": "kube-master",
			    "hosts": [
			        ""
			    ],
			    "key": {
			        "algo": "rsa",
			        "size": 2048
			    },
			    "names": [
			        {
			            "O": "system:masters"
			        }
			    ]
			}
            ```

            执行命令:

            ```
            cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client kube-master.json | cfssljson -bare master
            ```

            该证书用来管理集群使用，具有超级管理员的权限。关于kubernetes的API的授权相关内容参考：[Using RBAC Authorization](https://kubernetes.io/docs/admin/authorization/rbac/)。

2. 部署kubelet

	1. 下载kubelet的binary到/opt/bin/

        ```bash
        wegt https://storage.googleapis.com/kubernetes-release/release/v1.7.3/bin/linux/amd64/kubelet -O /opt/bin/kubelet && \ 
        chmod +x /opt/bin/kubelet
        ```

	2. 编辑systemd配置文件

		vim `/lib/systemd/system/kubelet.service`：

        ```
        [Unit]
        After=docker.service
        Requires=docker.service
        [Service]
        ExecStartPre=/bin/mkdir -p /etc/kubernetes/manifests
        ExecStart=/opt/bin/kubelet \
        --require-kubeconfig=true \
        --serialize-image-pulls=false \
        --kubeconfig=/etc/kubernetes/worker-kubeconfig.yaml \
        --network-plugin=cni \
        --allow-privileged=true \
        --container-runtime=docker \
        --docker=unix:///var/run/docker.sock \
        --pod-manifest-path=/etc/kubernetes/manifests \
        --hostname-override=10.22.108.20 \ #NOTE此处需要修改成所在机器IP
        --cluster-dns=10.99.66.2 \
        --cluster-domain=cluster.local \
        --max-pods=32 \
        --pod-infra-container-image=gcr.io/google_containers/pause-amd64:3.0
        Restart=always
        RestartSec=10
        [Install]
        WantedBy=multi-user.target
        ```

    3. 编辑kubelet访问apiserver的kubeconfig文件`/etc/kubernetes/worker-kubeconfig.yaml` ：

		```
		apiVersion: v1
		clusters:
		- cluster:
		    server: http://127.0.0.1:8080
		  name: default
		contexts:
		- context:
		    cluster: default
		    user: ""
		  name: default
		current-context: default
		kind: Config
		preferences: {}
		users: []
		```

3. 部署etcd

	etcd采用kubernetes的static pod的方式部署在k8s的master节点上。

	1. 将上面生成的etcd相关的证书包括`server.pem`、`server-key.pem`、`ca.pem` 拷贝到`/etc/etcd/ssl`目录下，  
        将`server{0..2}.pem`、`server{0..2}-key.pem`分别拷贝到对应三个机器的`/etc/etcd/ssl` 目录中，  
        改名成`memeber.pem`、`memeber-key.pem`。

	2. 编辑`/etc/kubernetes/manifests/etcd.yaml`

		```
		apiVersion: v1
		kind: Pod
		metadata:
		  name: etcd
		  namespace: kube-system
		  labels:
		    k8s-app: etcd
		spec:
		  hostNetwork: true
		  containers:
		  - name: etcd
		    image: quay.io/coreos/etcd:v3.2.3
		    command:
		    - etcd
		    - --name=infra0
		    - --data-dir=/var/etcd/data
		    - --cert-file=/etc/etcd/ssl/server.pem
		    - --key-file=/etc/etcd/ssl/server-key.pem
		    - --trusted-ca-file=/etc/etcd/ssl/ca.pem
		    - --peer-cert-file=/etc/etcd/ssl/member.pem
		    - --peer-key-file=/etc/etcd/ssl/member-key.pem
		    - --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem
		    - --client-cert-auth
		    - --peer-client-cert-auth
		    - --initial-advertise-peer-urls=https://10.22.108.20:2380
		    - --listen-peer-urls=https://10.22.108.20:2380
		    - --listen-client-urls=https://10.22.108.20:2379,https://127.0.0.1:2379
		    - --advertise-client-urls=https://10.22.108.20:2379
		    - --initial-cluster-token=6522f860-7ffb-495c-83ee-a0a4ebc39f90
		    - --initial-cluster=infra0=https://10.22.108.20:2380,infra1=https://10.22.108.79:2380,infra2=https://10.22.108.92:2380
		    - --initial-cluster-state=new
		    volumeMounts:
		    - mountPath: /etc/ssl/certs
		      name: ssl-certs-host
		      readOnly: true
		    - mountPath: /var/etcd/data
		      name: etcd-data
		    - mountPath: /etc/etcd/ssl
		      name: ssl-certs-etcd
		  volumes:
		  - hostPath:
		      path: /usr/share/ca-certificates
		    name: ssl-certs-host
		  - hostPath:
		      path: /var/etcd/data
		    name: etcd-data
		  - hostPath:
		      path: /etc/etcd/ssl
		    name: ssl-certs-etcd
		```

		需要注意的是：`--name=infra0`此处三个机器分别是infra0、infra1、infra2，需要上面各处出现的地址改成相应机器的IP，  
        `--initial-cluster-token` 此处使用`uuidgen`生成且各个机器保持一致，  
        `--initial-cluster=infra0=https://10.22.108.20:2380,infra1=https://10.22.108.79:2380,infra2=https://10.22.108.92:2380`此处各个机器的name和url保持对应。

4. 部署kube-apiserver

	1. 将上面生成的`apiserver.pem`、`apiserver-key.pem`、`ca.pem`、`ca-key.pem`拷贝到`/etc/kubernetes/ssl`文件夹中。   
        将`client.pem`、`client-key.pem` 拷贝到`/etc/etcd/ssl`文件夹中。

	2. 创建kube-apiserver使用的客户端token文件

		kubelet首次启动时向kube-apiserver发送TLS Bootstrapping请求，kube-apiserver 验证kubelet请求中的token是否与它配置的token.csv一致，  
        如果一致则自动为 kubelet生成证书和秘钥。

		```
		# head -c 16 /dev/urandom | od -An -t x | tr -d ' '
		7ebc01f3b35182b41acaeecf12857088

		# echo -n 7ebc01f3b35182b41acaeecf12857088,kubelet-bootstrap,10001,system:kubelet-bootstrap  > /etc/kubernetes/token.csv
		```

	3. 创建api-server启动的static pod，编辑`/etc/kubernetes/manifests/kube-apiserver.yaml`:

		```
		apiVersion: v1
		kind: Pod
		metadata:
		  name: kube-apiserver
		  namespace: kube-system
		spec:
		  hostNetwork: true
		  containers:
		  - name: kube-apiserver
		    image: quay.io/coreos/hyperkube:v1.7.3_coreos.0
		    command:
		    - /hyperkube
		    - apiserver
		    - --apiserver-count=3
		    - --bind-address=0.0.0.0
		    - --etcd-cafile=/etc/etcd/ssl/ca.pem
		    - --etcd-certfile=/etc/etcd/ssl/client.pem
		    - --etcd-keyfile=/etc/etcd/ssl/client-key.pem
		    - --etcd-servers=https://10.22.108.20:2379,https://10.22.108.79:2379,https://10.22.108.92:2379
		    - --allow-privileged=true
		    - --service-cluster-ip-range=10.99.66.0/23
		    - --secure-port=6443
		    - --advertise-address=10.22.108.20
		    - --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota
		    - --tls-cert-file=/etc/kubernetes/ssl/apiserver.pem
		    - --tls-private-key-file=/etc/kubernetes/ssl/apiserver-key.pem
		    - --client-ca-file=/etc/kubernetes/ssl/ca.pem
		    - --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem
		    - --authorization-mode=RBAC
		    - --experimental-bootstrap-token-auth
		    - --token-auth-file=/etc/kubernetes/token.csv
		    - --enable-swagger-ui=true
		    volumeMounts:
		    - mountPath: /etc/kubernetes/ssl
		      name: ssl-certs-kubernetes
		      readOnly: true
		    - mountPath: /etc/etcd/ssl
		      name: ssl-certs-etcd
		      readOnly: true
		    - mountPath: /etc/ssl/certs
		      name: ssl-certs-host
		      readOnly: true
		    - mountPath: /etc/kubernetes/token.csv
		      name: kubernetes-token
		  volumes:
		  - hostPath:
		      path: /etc/kubernetes/ssl
		    name: ssl-certs-kubernetes
		  - hostPath:
		      path: /etc/etcd/ssl
		    name: ssl-certs-etcd
		  - hostPath:
		      path: /usr/share/ca-certificates
		    name: ssl-certs-host
		  - hostPath:
		      path: /etc/kubernetes/token.csv
		    name: kubernetes-token
		```

		需要注意的是`--advertise-address=10.22.108.20` 此处需要改成对应机器的IP，`--authorization-mode=RBAC` 启用RBAC。

5. 部署kube-controller-manager

    编辑`/etc/kubernetes/manifests/kube-controll-manager.yaml`:

	```
	apiVersion: v1
	kind: Pod
	metadata:
	  name: kube-controller-manager
	  namespace: kube-system
	  labels:
	    k8s-app: kube-controller-manager
	spec:
	  hostNetwork: true
	  containers:
	  - name: kube-controller-manager
	    image: quay.io/coreos/hyperkube:v1.7.3_coreos.0
	    command:
	    - /hyperkube
	    - controller-manager
	    - --master=http://127.0.0.1:8080
	    - --leader-elect=true
	    - --cluster-name=kubernetes
	    - --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem
	    - --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem
	    - --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem
	    - --root-ca-file=/etc/kubernetes/ssl/ca.pem
	    - --allocate-node-cidrs=true
	    - --cluster-cidr=10.20.0.0/16
	    - --node-cidr-mask-size=27
	    livenessProbe:
	      httpGet:
	        host: 127.0.0.1
	        path: /healthz
	        port: 10252
	      initialDelaySeconds: 15
	      timeoutSeconds: 1
	    volumeMounts:
	    - mountPath: /etc/kubernetes/ssl
	      name: ssl-certs-kubernetes
	      readOnly: true
	    - mountPath: /etc/ssl/certs
	      name: ssl-certs-host
	      readOnly: true
	    - mountPath: /dev/
	      name: dev
	    - mountPath: /etc/ceph/
	      name: cephconfig
	  volumes:
	  - hostPath:
	      path: /etc/kubernetes/ssl
	    name: ssl-certs-kubernetes
	  - hostPath:
	      path: /usr/share/ca-certificates
	    name: ssl-certs-host
	  - hostPath:
	      path: /etc/ceph
	    name: cephconfig
	  - hostPath:
	      path: /dev/
	    name: dev
	```

	此处`--allocate-node-cidrs=true、--cluster-cidr=10.20.0.0/16` 将为每一个机器生成一个子网段，可以用来做IPAM。

6. 部署kube-scheduler

	编辑kube-scheduler的static pod的配置文件`/etc/kubernetes/kube-scheduler.yaml`

	```
	apiVersion: v1
	kind: Pod
	metadata:
	  name: kube-scheduler
	  namespace: kube-system
	  labels:
	    k8s-app: kube-scheduler
	spec:
	  hostNetwork: true
	  containers:
	  - name: kube-scheduler
	    image: quay.io/coreos/hyperkube:v1.7.3_coreos.0
	    command:
	    - /hyperkube
	    - scheduler
	    - --master=http://127.0.0.1:8080
	    - --leader-elect=true
	    livenessProbe:
	      httpGet:
	        host: 127.0.0.1
	        path: /healthz
	        port: 10251
	      initialDelaySeconds: 15
	      timeoutSeconds: 1
	```

7. 部署kube-proxy

	编辑`/etc/kubernetes/kube-proxy.yaml`:

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
	    - --master=http://127.0.0.1:8080
	    - --hostname-override=10.22.108.20
	    - --cluster-cidr=10.99.66.0/23
	    securityContext:
	      privileged: true
	    volumeMounts:
	    - mountPath: /etc/ssl/certs
	      name: ssl-certs-host
	      readOnly: true
	  volumes:
	  - hostPath:
	      path: /usr/share/ca-certificates
	    name: ssl-certs-host
	```

	需要把`--hostname-override=10.22.108.20` 改成对应的IP地址

8. 部署网络组建Flanneld

	flanneld可以[通过kubernetes的daemonset部署在容器里面](https://github.com/coreos/flannel/tree/master/Documentation)。
    为了避免因docker daemon重启带来的风险，我们把他部署在宿主机上，采用systemd unit启动。

	1. 生成flanneld访问etcd的证书。

		编辑flannel.json:

		```
		{
		    "CN": "flannel",
		    "hosts": [
		        ""
		    ],
		    "key": {
		        "algo": "rsa",
		        "size": 2048
		    },
		    "names": [
		        {
		            "O": "ETCD"
		        }
		    ]
		}

		```

		执行命令：

        ```
        cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client flannel.json | cfssljson -bare flannel
        ```

        将生成的证书`flannel.pem`、`flannel-key.pem`、`ca.pem` 放到`/etc/flannel/ssl`下面。

	2. 编辑flanneld的systemd unit,`/lib/systemd/system/flanneld.service`:

		```
		[Service]
		ExecStart=/opt/bin/flanneld \
		   -iface enp4s0f0 \
		   -etcd-cafile=/etc/flannel/ssl/ca.pem \
		   -etcd-certfile=/etc/flannel/ssl/flannel.pem \
		   -etcd-keyfile=/etc/flannel/ssl/flannel-key.pem \
		   -etcd-endpoints="https://10.22.108.20:2379,https://10.22.108.79:2379,https://10.22.108.92:2379" \
		   -etcd-prefix="/coreos.com/network" \
		   -ip-masq
		Restart=always
		RestartSec=10
		[Install]
		WantedBy=multi-user.target
		```

		需要把`-iface enp4s0f0` 改成对应的网卡名字，使用`systemctl start flanneld`启动flanneld。


	3. 在etcd中加入flanneld的配置项

		```
		etcdctl \
		  --endpoints=https://10.22.108.20:2379/ \
		  --ca-file=/etc/flannel/ssl/ca.pem  \
		  --cert-file=/etc/flannel/ssl/flannel.pem  \
		  --key-file=/etc/flannel/ssl/flannel-key.pem  \
		  set /coreos.com/network/config \
		  '{"Network":"10.100.0.0/16", "SubnetLen": 27, "Backend": {"Type": "vxlan"}}'
		```

9. 部署CNI

	1. 下载CNI

		```
		mkdir -p /opt/cni/bin/
		wget https://github.com/containernetworking/cni/releases/download/v0.6.0/cni-amd64-v0.6.0.tgz -O - |tar -zxpvf - -C /opt/cni/bin/
		```

	2. 编辑CNI配置

		编辑`/etc/cni/net.d/10-containers.conf`:

		```
		{
		    "name": "cbr0",
		    "type": "flannel",
		    "delegate": {
		        "isDefaultGateway": true
		    }
		}
		```

至此master节点就配置完成了。使用`systemctl enable kubelet flanneld` 确保服务开机自启。
如果遇到无法获取gcr.io的镜像，可以绑定hosts `216.58.220.192 gcr.io`。
