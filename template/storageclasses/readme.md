# StorageClass比较麻烦，这里就说明下

## [StorageClass](https://kubernetes.io/zh-cn/docs/concepts/storage/storage-classes/)

## StorageClass 资源
每个 StorageClass 都包含 provisioner、parameters 和 reclaimPolicy 字段， 这些字段会在 StorageClass 需要动态制备 PersistentVolume 时会使用到。

StorageClass 对象的命名很重要，用户使用这个命名来请求生成一个特定的类。 当创建 StorageClass 对象时，管理员设置 StorageClass 对象的命名和其他参数，一旦创建了对象就不能再对其更新。

管理员可以为没有申请绑定到特定 StorageClass 的 PVC 指定一个默认的存储类： 更多详情请参阅

[PersistentVolumeClaim 章节](https://kubernetes.io/zh-cn/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
  - debug
volumeBindingMode: Immediate
```

## 官网NFS 例子
官网的例子比较粗燥，笔者也弄了一个比较详细的例子
```shell
# 官网的nfs-storageClass例子
cat > storageClass.yaml <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: example-nfs
provisioner: example.com/external-nfs
parameters:
  server: nfs-server.example.com
  path: /share
  readOnly: "false"
EOF
```
* server：NFS 服务器的主机名或 IP 地址。
* path：NFS 服务器导出的路径。
* readOnly：是否将存储挂载为只读的标志（默认为 false）。


## 笔者示例
```shell
# 1、新增 serviceAccount nfs-client-provisioner 作为 NFS provisioner的权限来源
# 2、在k8s中有赋予 serviceAccount 足够的权限处理跟 StorageClass && PersistentVolumeClaim 相关的内容
# （通过 Role + RoleBinDing + ClusterRole + ClusterRoleBinding）
# 创建rbca权限
cat > rbca.yaml << EOF
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io

---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]

---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io

EOF

## 应用资源
kubectl apply -f rbca.yaml

```

## 安装 NFS provisioner
* 1、在 NFS share 中产生 volume(directory)
* 2、新增 PV
* 3、告知 PVC 已经完成PV的设定

```shell
# 创建nfs-deployment
cat > nfs-deploy.yaml << EOF
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: my-nfs-provisioner  ## storageClass需要通过这里这个名称来绑定 provisioner
            - name: NFS_SERVER
              value: 10.1.2.3
            - name: NFS_PATH
              value: /var/k8s-nfs-share
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.1.2.3
            path: /var/k8s-nfs-share

EOF

# 创建 StorageClass
cat > storageClass.yaml << EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # 开启默认使用这个storageClass策略来创建pv申领逻辑卷
  name: my-nfs-storage
provisioner: my-nfs-provisioner
parameters:
  archiveOnDelete: "false"

EOF

## 应用资源
kubectl apply -f nfs-deploy.yaml -f storageClass.yaml

```

## 验证
创建一个pvc来验证逻辑卷的申领

```shell
# 创建一个pvc
cat > pvc << EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: "my-nfs-storage"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi

EOF

# 查看创建结果
$ kubectl get pvc
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
test-claim   Bound    pvc-8e95052b-9dda-e958-11e8-9a0asdf90dsf   1Mi        RWX            my-nfs-storage        5s

# 檢視 PV
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                                 STORAGECLASS          REASON   AGE
pvc-8e95052b-9dda-e958-11e8-9a0asdf90dsf   1Gi        RWX            Delete           Bound    default/test-claim                                                    managed-nfs-storage            1m46s

```


## [更多参考案例](https://github.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner/tree/master/deploy/kubernetes)



