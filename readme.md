一、yaml文件列表
1.针对cvallance/mongo-k8s-sidecar的错误，需要绑定serviceaccount default设置的

default-clusterrolebinding.yaml
kind: ClusterRoleBinding
metadata:
  name: default-view
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
  - kind: ServiceAccount
    name: default
    namespace: dev
2.部署mongo statefulset

mongo-deployment.yaml
kind: Service
metadata:
  name: mongodb
spec:
  type: NodePort
  ports:
  - port: 27017
    targetPort: 27017
  selector:
    k8s-app: mongodb
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: mongodb
  replicas: 3
  selector:
    matchLabels:
      k8s-app: mongodb
  template:
    metadata:
      labels:
        k8s-app: mongodb
        role: mongo
        environment: test
    spec:
      containers:
      - name: mongo
        image: mongo:3.4.4
        imagePullPolicy: IfNotPresent
        command:
        - mongod
        - "--replSet"
        - rs0
        - "--bind_ip"
        - 0.0.0.0
        - "--smallfiles"
        - "--noprealloc"
        ports:
        - containerPort:  27017
          protocol: TCP
        volumeMounts:
        - name: pvc
          mountPath: /data/mongo
      - name: mongo-sidecar
        image:  cvallance/mongo-k8s-sidecar:latest
        imagePullPolicy: IfNotPresent
        env:
        - name: MONGO_SIDECAR_POD_LABELS
          value: "role=mongo,environment=test"

  volumeClaimTemplates:
  - metadata:
      name: pvc
      annotations:
        volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
    spec:
      accessModes: [ "ReadWriteMany" ]
      storageClassName:  managed-nfs-storage
      resources:
        requests:
          storage: 5Gi
3.部署sotrageclass

mongo-storageclass.yaml
kind: StorageClass
metadata:
  name: managed-nfs-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: fuseim.pri/ifs
reclaimPolicy: Retain
4.这说说明一下nfs-subdir-external-provisioner 需要4.0以上版本，否则会出现无法自动创建pvc，导致pv绑定失败

nfs-client-provisioner-deploy.yaml
kind: Deployment
metadata:
  name: nfs-client-provisioner
  namespace: dev
  labels:
    app: nfs-client-provisioner
spec:
  selector:
    matchLabels:
      app: nfs-client-provisioner
  replicas: 1
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: sa-nfs-client-provisioner
      containers:
      - name:  nfs-client-provisioner
        image:  registry.cn-beijing.aliyuncs.com/mydlq/nfs-subdir-external-provisioner:v4.0.0
        imagePullPolicy: Never
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 100m
            memory: 100Mi        
        volumeMounts:
        - name: nfs-client-root
          mountPath: /persistentvolumes
        - name: localtime
          mountPath: /etc/localtime
        env:
        - name: PROVISIONER_NAME
          value: fuseim.pri/ifs
        - name: NFS_SERVER
          value: 192.168.15.103
        - name: NFS_PATH
          value: /data/mongo
      volumes:
      - name: nfs-client-root
        nfs:
          server: 192.168.15.103
          path: /data/mongo
      - name: localtime
        hostPath:
          path: /usr/share/zoneinfo/Asia/Shanghai
5.nfs-client-provisioner的RBAC设置，权限分配不做过多说明

nfs-client-provisioner-rbac.yaml
apiVersion: v1
metadata:
  name: sa-nfs-client-provisioner
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: role-nfs-client-provisioner
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rb-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: sa-nfs-client-provisioner
    namespace: dev
roleRef:
  kind: Role
  name: role-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io    
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cr-nfs-client-provisioner
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
  name: crb-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: sa-nfs-client-provisioner
    namespace: dev
roleRef:
  kind: ClusterRole
  name: cr-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
二、所需镜像
mongo:3.4.4
registry.cn-beijing.aliyuncs.com/mydlq/nfs-subdir-external-provisioner:v4.0.0
cvallance/mongo-k8s-sidecar:latest
三、系统及服务
操作系统：Centos7.9
安装包：nfs-utils ---直接yum安装即可，或者去清华下载包也行
四、部署须知：
本yaml文件已经指定namespace: dev,如果想要修改namespace请使用 sed -i 's#namespace: dev#自定义namespace#g' *即可
