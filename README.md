### 一、克隆glusterfs仓库，并在所有glusterfs节点安装.

```javascript
git clone https://github.com/luo964973791/glusterfs.git
cd glusterfs
yum install ./*.rpm -y
```

### 二、工作节点打标签

```javascript
kubectl label node node1 storagenode=glusterfs
kubectl label node node2 storagenode=glusterfs
kubectl label node node3 storagenode=glusterfs
```

### 三、加载模块

```javascript
# 加载模块
modprobe dm_snapshot
modprobe dm_mirror
modprobe dm_thin_pool
# 验证模块是否加载成功
lsmod | grep dm_snapshot
lsmod | grep dm_mirror
lsmod | grep dm_thin_pool
```

### 四、解压安装包

```javascript
cd glusterfs
tar zxvf heketi-v10.4.0-release-10.linux.amd64.tar.gz
cd heketi-client/share/heketi/kubernetes/
vi heketi.json
"admin": {
      "key": "admin"
    },
    "_user": "User only has access to /volumes endpoint",
    "user": {
      "key": "admin"
    }



kubectl apply -f glusterfs-daemonset.json 
kubectl apply -f heketi-service-account.json
kubectl create clusterrolebinding heketi-gluster-admin --clusterrole=cluster-admin --serviceaccount=default:heketi-service-account
kubectl create secret generic heketi-config-secret --from-file=./heketi.json
kubectl create -f heketi-bootstrap.json
kubectl get pods
cp /root/heketi-client/bin/heketi-cli /usr/local/bin/
```

### 五、配置topology-sample.json

```javascript
[root@node1 ~]# cat heketi-client/share/heketi/kubernetes/topology-sample.json 
{
  "clusters": [
    {
      "nodes": [
        {
          "node": {
            "hostnames": {
              "manage": [
                "node1"
              ],
              "storage": [
                "172.27.0.3"
              ]
            },
            "zone": 1
          },
          "devices": [
            {
              "name": "/dev/sdb",
              "destroydata": true
            }
          ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "node2"
              ],
              "storage": [
                "172.27.0.4"
              ]
            },
            "zone": 1
          },
          "devices": [
            {
              "name": "/dev/sdb",
              "destroydata": true
            }
          ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "node3"
              ],
              "storage": [
                "172.27.0.5"
              ]
            },
            "zone": 1
          },
          "devices": [
            {
              "name": "/dev/sdb",
              "destroydata": true
            }
          ]
        }
      ]
    }
  ]
}
```

### 六、获取当前heketi的ClusterIP,用heketi创建glusterfs集群

```javascript
kubectl get svc|grep heketi
curl http://x.x.x.x:8080/hello
export HEKETI_CLI_SERVER=http://10.111.162.155:8080
echo $HEKETI_CLI_SERVER
heketi-cli -s $HEKETI_CLI_SERVER --user admin --secret 'admin' topology load --json=topology-sample.json
heketi-cli -s $HEKETI_CLI_SERVER --user admin --secret 'admin' topology info
```

### 七、获取k8s资源文件

```javascript
heketi-cli -s $HEKETI_CLI_SERVER --user admin --secret 'admin' setup-openshift-heketi-storage Saving heketi-storage.json
kubectl apply -f heketi-storage.json
kubectl delete all,svc,jobs,deployment,secret --selector="deploy-heketi"
kubectl apply -f heketi-deployment.json
kubectl get svc|grep heketi
curl http://x.x.x.x:8080/hello
export HEKETI_CLI_SERVER=http://10.111.162.155:8080
echo $HEKETI_CLI_SERVER
heketi-cli -s $HEKETI_CLI_SERVER --user admin --secret 'admin' topology info
```

### 八、创建 storageclass-gfs-heketi.yaml

```javascript
apiVersion: v1
kind: Secret
metadata:
  name: heketi-secret
  namespace: kube-system
type: kubernetes.io/glusterfs
data:
  key: "YWRtaW4="    #请替换为您自己的密钥。Base64 编码
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.beta.kubernetes.io/is-default-class: "true"
    storageclass.kubesphere.io/supported-access-modes: '["ReadWriteOnce","ReadOnlyMany","ReadWriteMany"]'
  name: glusterfs
parameters:
  clusterid: "86b78d5c5eb4fda20f5626188d46b77f"    #请替换为您自己的 GlusterFS 集群 ID。
  gidMax: "50000"
  gidMin: "40000"
  restauthenabled: "true"
  resturl: "http://10.101.246.205:8080"    #Gluster REST 服务/Heketi 服务 URL 可按需供应 gluster 存储卷。请替换为您自己的 URL。
  restuser: admin
  secretName: heketi-secret
  secretNamespace: kube-system
  volumetype: "replicate:3"    #请替换为您自己的存储卷类型。
provisioner: kubernetes.io/glusterfs
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
kubectl apply -f storageclass-gfs-heketi.yaml
```

### 九、测试

```javascript
apiVersion: v1
kind: Pod
metadata:
  name: pod-use-pvc
spec:
  containers:
  - name: pod-use-pvc
    image: busybox:1.28.3
    command:
      - sleep
      - "3600"
    volumeMounts:
    - name: gluster-volume
      mountPath: "/pv-data"
      readOnly: false
  volumes:
  - name: gluster-volume
    persistentVolumeClaim:
      claimName: pvc-gluster-heketi

---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-gluster-heketi
spec:
  accessModes: [ "ReadWriteOnce" ]
  storageClassName: "glusterfs"
  resources:
    requests:
      storage: 1Gi
```

