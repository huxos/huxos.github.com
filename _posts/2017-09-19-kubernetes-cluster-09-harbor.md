---
layout: post
title: kubernetes集群搭建（9. harbor）
description: kubernetes集群搭建
categories:
  - Kubernetes
  - Harbor
---

### kubernetes集群搭建（9. 镜像仓库harbor）

[Harbor](https://github.com/vmware/harbor)是VMware开发的企业级docker镜像仓库，官方提供了基于docker-compose的方式部署。  
花了好些时间将其conpose的编排模版转换成k8s的调度模版方便部署到k8s集群中，项目地址为: [https://github.com/huxos/harbor-kubernetes](https://github.com/huxos/harbor-kubernetes)。


部署架构：

- 把harbor的mysql组件部署到单独的一个Pod中，采用ceph的rbd作为数据库的存储。
- 新增redis做session共享。
- 采用ceph的rgw作为镜像的后端存储。
- 其余组件（包括adminserver、jobservice、ui、nginx、registry）部署到同一个Pod中，使用127.0.0.1通讯。

这样部署可以通过k8s使得harbor灵活的缩放。


#### 部署步骤

1.	部署ceph-rgw并添加访问ceph的用户

	由于我们使用ceph的rgw作为registry的后端，首先参照之前部署ceph的步骤[kubernetes集群搭建（2. Ceph）](http://huxos.me/kubernetes/ceph/2017/09/19/kubernetes-cluster-02-ceph.html)
    在kube-system-2 kube-system-4上面部署`ceph-rgw`

	```
	ceph-deploy rgw create kube-system-2 kube-system-4
	```

	添加访问rgw的账户：

	```
	##创建用户
	radosgw-admin user create --uid=registry --display-name="ceph rgw docker registry user"

	#registry支持ceph的swift协议，创建swift子账号
	radosgw-admin subuser create --uid registry --subuser=registry:swift --access=full --secret=secretkey --key-type=swift

	#创建secret key
	radosgw-admin key create --subuser=registry:swift --key-type=swift --gen-secret

	{
	    "user_id": "registry",
	    "display_name": "ceph rgw docker registry user",
	    "email": "",
	    "suspended": 0,
	    "max_buckets": 1000,
	    "auid": 0,
	    "subusers": [
	        {
	            "id": "registry:swift",
	            "permissions": "full-control"
	        }
	    ],
	    "keys": [
	        {
	            "user": "registry",
	            "access_key": "********************",
	            "secret_key": "****************************************"
	        }
	    ],
	    "swift_keys": [
	        {
	            "user": "registry:swift",
	            "secret_key": "e***************************************"
	        }
	    ],
	    "caps": [],
	    "op_mask": "read, write, delete",
	    "default_placement": "",
	    "placement_tags": [],
	    "bucket_quota": {
	        "enabled": false,
	        "max_size_kb": -1,
	        "max_objects": -1
	    },
	    "user_quota": {
	        "enabled": false,
	        "max_size_kb": -1,
	        "max_objects": -1
	    },
	    "temp_url_keys": []
	}
	```
	记录好上面的swift_keys里面的secret_key: `e*************************************** `

2.	生成registry 使用的证书

	参照前篇[kubernetes集群搭建（3. Kube master）](http://huxos.me/kubernetes/2017/09/19/kubernetes-cluster-03-kube-master.html)生成证书，供给harbor使用。

	编辑harbor.json

	```
	{
	    "CN": "harbor",
	    "hosts": [
	        "registry.example.com",
	        "quay.io",
	        "gcr.io"
	    ],
	    "key": {
	        "algo": "rsa",
	        "size": 2048
	    },
	    "names": [
	        {
	            "O": "REGISTRY"
	        }
	    ]
	}
	```

	使用命令: `cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server harbor.json | cfssljson -bare harbor` 生成证书。  
    这里的证书需要签名访问harbor使用的域名，根据实际情况替换掉`registry.example.com`

	创建harbor部署的namespace:

	```
	kubectl create ns registry
	```

	作为secret导入证书：

	```
	kubectl create secret tls harbor --key harbor-key.pem --cert harbor.pem
	```

3. 部署harbor组件

	1.	harbor的部署模版以及提交到git仓库中，首先将其clone下来

		```
		git clone https://github.com/huxos/harbor-kubernetes.git
		```

	2.	部署mysql组件

        参照前篇为导入ceph的key到registry中[kubernetes集群搭建（2. Ceph）](http://huxos.me/kubernetes/ceph/2017/09/19/kubernetes-cluster-02-ceph.html)

		```
		#导入ceph的key到registry的namespace中
		kubectl create secret generic ceph-secret-kube --type="kubernetes.io/rbd" \
            --from-literal=key=`ceph --cluster ceph auth get-key client.kube` --namespace=registry

		# mysql使用的rbd
		kubectl create -f pvc/mysql-data.yaml

		# 导入mysql的环境变量
		kubectl create -f cm/db-env.yaml

		# 部署mysql
		kubectl create -f mysql.deploy.yaml

		# svc
		kubectl create -f svc/mysql.yaml
		```

	3. 部署redis

		```
		kubectl create -f redis.deploy.yaml
		kubectl create -f svc/redis.yaml
		```

	4.	部署harbor

		首先编辑cm下面的configMap文件、将`cm/registry.yaml`里面的harbor的访问地址`registry.example.com`根据实际情况替换掉，配置的ceph rgw的访问地址和密码。

		```
		#导入configmap
		#adminserver
		kubectl create -f cm/adminserver.yaml -f cm/adminserver-env.yaml
		#jobservice
		kubectl create -f cm/jobservice.yaml -f cm/jobservice-env.yaml
		#registry
		kubectl create -f cm/registry.yaml
		#ui
		kubectl create -f cm/ui-env.yaml -f cm/ui.yaml
		#nginx
		kubectl create -f cm/nginx.yaml

		#部署harbor
		kubectl create -f harbor.deploy.yaml
		```
4. 配置docker

    由于访问reigstry的证书是自签名的需要将ca放到docker的ca目录中  

	将ca文件复制到 `/etc/docker/<registry的访问域名>／ca.crt`
