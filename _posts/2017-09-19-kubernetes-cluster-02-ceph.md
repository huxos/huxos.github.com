---
layout: post
title: kubernetes集群搭建（2. Ceph）
description: kubernetes集群搭建
categories:
  - Kubernetes
  - Ceph
---

本例中使用5个机器部署ceph`/dev/sdb`作为osd使用的分区，使用ceph-deploy搭建了简易的ceph集群。     
该Ceph集群用来存储包括docker镜像仓库，prometheus的监控数据等。  
使用kubernetes的StorageClass对接ceph，使得ceph的rdb可以作为数据卷挂载给Pod使用。

1. 安装ceph-deploy

    ```
    apt-get install python-pip virtualenv
    virtualenv ~/.env
    source ~/.env/bin/activate
    pip install ceph-deploy --index-url https://pypi.tuna.tsinghua.edu.cn/simple
    ```

2. 使用ceph-deploy安装ceph

    `注意：` 安装ceph要求mon节点时间要同步，可以先配置好ntpd做时间同步。采用命令`timedatectl status`或者`ntpq -p`查看同步状态。

    1. 首先编辑5台机器的hosts文件加入如下内容

        ```
        10.22.108.20 kube-system-0
        10.22.108.21 kube-system-1
        10.22.108.22 kube-system-2
        10.22.108.23 kube-system-3
        10.22.108.24 kube-system-4
        ```

    2. 配置好ssh验证

        编辑`~/.ssh/config`加入如下内容

        ```
        StrictHostKeyChecking no
        UserKnownHostsFile /dev/null

        Host kube-system-*
            Port 22
            User root

        Host 10.22.108.*
            Port 22
            User root
        ```

        配置ssh公钥登录并执行`ssh kube-system-1`测试是否能正常登录。

    3. 安装ceph软件包

        执行命令：`ceph-deploy install --repo-url https://mirrors.ustc.edu.cn/ceph/debian-jewel kubernetes-{0..4}`

    4. 部署ceph mon节点

        执行命令：  
        `ceph-deploy --cluster ceph new kube-system-{0..4} --public-network=10.22.108.0/23 --private-network=10.22.108.0/23`

    5. 部署osd

        执行命令：`ceph-deploy osd create kube-system-{0..4}:/dev/sdb`创建osd,   
        然后在执行:`ceph-deploy osd activate kube-system-{0..4}`应用osd。

    6. 部署管理节点

        执行命令: `ceph-deploy admin kube-system-{0..4}`，执行`ceph -s `查看集群运行状态。


3. 创建供给kubernetes使用的pool

    执行命令`ceph --cluster ceph osd create kube 1024 1024`


4. 导入ceph的key到kubernetes集群中去

    将ceph的admin-key到k8s中去，作为secret导入kube-system这个namespace中

    ```bash
    kubectl create secret generic ceph-secret --type="kubernetes.io/rbd" \
        --from-literal=key=`ceph --cluster ceph auth get-key client.admin`  --namespace=kube-system
    ```

    创建client.kube并将key导入到各个需要使用rbd的namespaces中

    ```bash
    ceph --cluster ceph auth get-or-create client.kube mon 'allow r' osd 'allow rwx pool=kube'
    kubectl create secret generic ceph-secret-kube --type="kubernetes.io/rbd" \
        --from-literal=key=`ceph --cluster ceph auth get-key client.kube` --namespace=default
    ```

5. 创建StorageClass

    编辑rbd-storgaeclass.yaml：

    ```yaml
    kind: StorageClass
    metadata:
      name: standard
      annotations:
        storageclass.kubernetes.io/is-default-class: "true"
    provisioner: kubernetes.io/rbd
    parameters:
      monitors: 10.22.108.20:6789,10.22.108.21:6789,10.22.108.22:6789,10.22.108.23:6789,10.22.108.24:6789
      adminId: admin
      adminSecretName: ceph-secret
      adminSecretNamespace: "kube-system"
      pool: kube
      userId: kube
      userSecretName: ceph-secret-kube
    ```

6. 使用ceph

    首先需要在各个k8s节点中安装`ceph-common`,如果kube-controller-manager部署在容器中，需要基础镜像安装了ceph-common。  
    coreos提供的hyperkube的镜像`quay.io/coreos/hyperkube:v1.7.3_coreos.0`就包含了ceph-common，可以考虑使用此镜像部署kubernetes。

    创建kubernetes的pvc

    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: metadata-storage
      namespace: default
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 100Gi
      storageClassName: standard
    ```

    创建好之后k8s会根据kubernetes会根据pvc模版创建相对应的pv，使用`kubectl get pv`查看。  
    使用`kubectl describe pv <PVNAME>`可以获取到pv对应的ceph/rbd。

    ```bash
    # kubectl describe pv pvc-01b09435-7b49-11e7-8938-1866da9e3d4b
    
    Name:  pvc-01b09435-7b49-11e7-8938-1866da9e3d4b
    Labels:  <none>
    Annotations:  kubernetes.io/createdby=rbd-dynamic-provisioner
      pv.kubernetes.io/bound-by-controller=yes
      pv.kubernetes.io/provisioned-by=kubernetes.io/rbd
    StorageClass:  standard
    Status:  Bound
    Claim:  default/metadata-storage
    Reclaim Policy:  Delete
    Access Modes:  RWO
    Capacity:  100Gi
    Message:
    Source:
      Type:  RBD (a Rados Block Device mount on the host that shares a pod's lifetime)
      CephMonitors:  [10.22.108.20:6789 10.22.108.21:6789 10.22.108.22:6789 10.22.108.23:6789 10.22.108.24:6789]
      RBDImage:  kubernetes-dynamic-pvc-01b2719f-7b49-11e7-8bd4-1866da9e3d4b
      FSType:
      RBDPool:  kube
      RadosUser:  kube
      Keyring:  /etc/ceph/keyring
      SecretRef:  &{ceph-secret-kube}
      ReadOnly:  false
    Events:    <none>
    
    # rbd ls kube |grep kubernetes-dynamic-pvc-01b2719f-7b49-11e7-8bd4-1866da9e3d4b
    kubernetes-dynamic-pvc-01b2719f-7b49-11e7-8bd4-1866da9e3d4b
    ```

    然后创建pod使用pv

    ```yaml
    kind: Deployment
    metadata:
    labels:
      run: metadata-server
    name: metadata-server
    namespace: default
    spec:
      replicas: 1
      selector:
        matchLabels:
          run: metadata-server
      strategy:
        rollingUpdate:
          maxSurge: 1
          maxUnavailable: 1
        type: RollingUpdate
      template:
        metadata:
          labels:
            run: metadata-server
        spec:
          containers:
          - image: nginx:alpine
            name: metadata-server
            volumeMounts:
            - mountPath: /usr/share/nginx/html
              name: metadata-storage
          volumes:
          - name: metadata-storage
            persistentVolumeClaim:
              claimName: metadata-storage
    ```

    等到pod启动成功，到pod调度的机器上面可以看到对应的rbd已经格式化并挂载好了。

    ```bash
    mount | grep pvc-01b09435-7b49-11e7-8938-1866da9e3d4b
    /dev/rbd2 on /var/lib/kubelet/pods/41104258-88df-11e7-a533-1866daa4fa2f/volumes/kubernetes.io~rbd/pvc-01b09435-7b49-11e7-8938-1866da9e3d4b type ext4 (rw,relatime,stripe=1024,data=ordered)
    ```

    如果应用采用statefulset的方式部署，则可以不用先创建pvc，直接在statefulset的模版里面指定volumeClaimTemplates:

    ```yaml
    volumeClaimTemplates:
    - metadata:
        name: prometheus-k8s-db
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Ti
    ```

至此ceph部署和在kubernetes 中的应用就介绍完了。
