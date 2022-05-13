### 一、yum安装gluster + heketi

```javascript
yum -y install attr psmisc rpcbind openssl openssl-devel
yum install centos-release-gluster9 -y
yum install glusterfs-server glusterfs-client heketi-client heketi -y
systemctl start glusterd glusterfsd && systemctl enable glusterd glusterfsd
```

### 二、多台集群添加，只在一台机器执行

```javascript
gluster peer probe node2
gluster peer probe node3
gluster pool list
systemctl start heketi.service && systemctl enable heketi.service
```

### 三、打通所有节点密钥

```javascript
sudo ssh-keygen -f /etc/heketi/heketi_key -t rsa -N ''
sudo chown heketi:heketi /etc/heketi/heketi_key*
ssh-copy-id -i /etc/heketi/heketi_key.pub root@node1
ssh-copy-id -i /etc/heketi/heketi_key.pub root@node2
ssh-copy-id -i /etc/heketi/heketi_key.pub root@node3




cat /etc/heketi/heketi.json
{
  "_port_comment": "Heketi Server Port Number",
  "port": "8080",

  "_use_auth": "Enable JWT authorization. Please enable for deployment",
  "use_auth": true,

  "_jwt": "Private keys for access",
  "jwt": {
    "_admin": "Admin has access to all APIs",
    "admin": {
      "key": "admin"
    },
    "_user": "User only has access to /volumes endpoint",
    "user": {
      "key": "admin"
    }
  },

  "_glusterfs_comment": "GlusterFS Configuration",
  "glusterfs": {
    "_executor_comment": [
      "Execute plugin. Possible choices: mock, ssh",
      "mock: This setting is used for testing and development.",
      "      It will not send commands to any node.",
      "ssh:  This setting will notify Heketi to ssh to the nodes.",
      "      It will need the values in sshexec to be configured.",
      "kubernetes: Communicate with GlusterFS containers over",
      "            Kubernetes exec api."
    ],
    "executor": "ssh",

    "_sshexec_comment": "SSH username and private key file information",
    "sshexec": {
      "keyfile": "/etc/heketi/heketi_key",
      "user": "root",
      "port": "22",
      "fstab": "/etc/fstab"
    },

    "_kubeexec_comment": "Kubernetes configuration",
    "kubeexec": {
      "host" :"https://kubernetes.host:8443",
      "cert" : "/path/to/crt.file",
      "insecure": false,
      "user": "kubernetes username",
      "password": "password for kubernetes user",
      "namespace": "OpenShift project or Kubernetes namespace",
      "fstab": "Optional: Specify fstab file on node.  Default is /etc/fstab"
    },

    "_db_comment": "Database file name",
    "db": "/var/lib/heketi/heketi.db",

    "_loglevel_comment": [
      "Set log level. Choices are:",
      "  none, critical, error, warning, info, debug",
      "Default is warning"
    ],
    "loglevel" : "debug"
  }
}
```

### 四、在heketi节点测试.

```javascript
systemctl restart heketi
systemctl status heketi
curl http://localhost:8080/hello
export HEKETI_CLI_SERVER=http://localhost:8080



vi /usr/share/heketi/topology-sample.json
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


#构建集群
heketi-cli -s $HEKETI_CLI_SERVER --user admin --secret 'admin' topology load --json=/usr/share/heketi/topology-sample.json

#测试挂载是否成功.
heketi-cli volume create --size=10 --durability=none --user "admin" --secret "admin"
```

### 五、创建StorageClass

```javascript
vi storageclass-gfs-heketi-distributed.yaml 
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: glusterfs
provisioner: kubernetes.io/glusterfs
reclaimPolicy: Retain
parameters:
  resturl: "http://172.27.0.3:8080"
  restauthenabled: "true"
  restuser: "admin"
  restuserkey: "admin"
  gidMin: "40000"
  gidMax: "50000"
  volumetype: "replicate:3"
allowVolumeExpansion: true
volumeBindingMode: Immediate
```

### 六、创建pvc

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

### 七、客户端挂载

```javascript
[root@node1 localhost]# gluster volume info | grep -E "Brick1|Volume Name" 
Volume Name: vol_605a5942ce7fb06162eb7ea1c7a5bab9
Brick1: 172.27.0.5:/var/lib/heketi/mounts/vg_f402e3f2717c0086255c5a11123d205a/brick_9901daf53868ac39a910d32a428e8e8f/brick
mount -t glusterfs 172.27.0.5:vol_605a5942ce7fb06162eb7ea1c7a5bab9 /mnt
```

