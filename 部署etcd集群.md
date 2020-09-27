[TOC]

# 部署etcd集群

## 1、cfssl证书工具准备

使用`cfssl`来生成自签证书，先下载`cfssl`工具

```bash
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64   #传入json文件生成
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64  #查看证书信息

```

放到`/usr/local/bin/`目录下，以便cfssl能直接执行

```bash
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo

```

同步虚拟机时间

```
ntpdate time.window.com
```

### 1.1 生成证书

创建以下三个文件：

1，**生成ca证书csr的json配置文件**

- `cat ca-csr.json*`

```json
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
```

2，设置ca机构的属性文件

- `ca-config.json`

```json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "www": {
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

3,生成etcd的server的证书文件

- `server-csr.json`
- hosts:包含etcd集群的ip地址

```json
{
    "CN": "etcd",
    "hosts": [
        "172.22.1.11",
        "172.22.1.21",
        "172.22.1.22"
        ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing"
        }
    ]
}
```

## 2,生成证书

使用sfssl方式，通过json文件配置要生成的证书

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca –
#用cfssl生成证书 初始化 ca机构（输出json）
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server
# ls *pem
ca-key.pem  ca.pem  server-key.pem  server.pem
```

证书这块知道怎么生成、怎么用即可，建议暂时不必过多研究。

### 2.1部署Etcd

二进制包下载地址：<https://github.com/coreos/etcd/releases/tag/v3.2.12>

以下部署步骤在规划的三个etcd节点操作一样，唯一不同的是etcd配置文件中的服务器P要写当前的：

### 2.2在master01(172.22.1.11)上面操作

- 解压二进制包：

```bash
mkdir /opt/etcd/{bin,cfg,ssl} -p
tar zxvf etcd-v3.2.12-linux-amd64.tar.gz
mv etcd-v3.2.12-linux-amd64/{etcd,etcdctl} /opt/etcd/bin/
```

- 创建etcd配置文件：

```
# cat /opt/etcd/cfg/etcd.conf   
#[Member]
ETCD_NAME="master01"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://172.22.1.11:2380"
ETCD_LISTEN_CLIENT_URLS="https://172.22.1.11:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://172.22.1.11:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://172.22.1.11:2379"
ETCD_INITIAL_CLUSTER="master01=https://172.22.1.11:2380,node01=https://172.22.1.21:2380,node02=https://172.22.1.22:2380" 
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```

- ETCD_NAME 节点名称
- ETCD_DATA_DIR 数据目录
- ETCD_LISTEN_PEER_URLS 集群之间通信监听地址 (master01 ip)
- ETCD_LISTEN_CLIENT_URLS 客户端访问监听地址(可以读写data，相当于mysql的3306) master01 ip
- ETCD_INITIAL_ADVERTISE_PEER_URLS 集群通告地址
- ETCD_ADVERTISE_CLIENT_URLS 客户端通告地址 
- ETCD_INITIAL_CLUSTER 集群节点地址 (master01 node01 node02的ip)
- ETCD_INITIAL_CLUSTER_TOKEN 集群Token
- ETCD_INITIAL_CLUSTER_STATE 加入集群的当前状态，new是新集群，existing表示加入已有集群

### 2.3systemd管理etcd

```
# cat /usr/lib/systemd/system/etcd.service 
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/opt/etcd/cfg/etcd.conf
ExecStart=/opt/etcd/bin/etcd \
        --cert-file=/opt/etcd/ssl/server.pem \
        --key-file=/opt/etcd/ssl/server-key.pem \
        --peer-cert-file=/opt/etcd/ssl/server.pem \
        --peer-key-file=/opt/etcd/ssl/server-key.pem \
        --trusted-ca-file=/opt/etcd/ssl/ca.pem \
        --peer-trusted-ca-file=/opt/etcd/ssl/ca.pem
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

```

### 2.4提示错误

```
conflicting environment variable "ETCD_NAME" is shadowed by corresponding command-line flag (either unset environment variable or disable flag)
```

- 原因：ETCD3.4版本会自动读取环境变量的参数，所以EnvironmentFile文件中有的参数，不需要再次在ExecStart启动参数中添加，二选一，如同时配置，会触发以下类似报错是因。

#### 解决方法：

删除etcd.service中引用的变量，或者删除`EnvironmentFile=/opt/etcd/cfg/etcd.conf`这一行，并且不使用conf引用变量

```
# cat /usr/lib/systemd/system/etcd.service 
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/opt/etcd/cfg/etcd.conf
ExecStart=/opt/etcd/bin/etcd \
        
        --data-dir=${ETCD_DATA_DIR} \
        --listen-peer-urls=${ETCD_LISTEN_PEER_URLS} \
        --listen-client-urls=${ETCD_LISTEN_CLIENT_URLS},http://127.0.0.1:2379 \
        --advertise-client-urls=${ETCD_ADVERTISE_CLIENT_URLS} \
        --initial-advertise-peer-urls=${ETCD_INITIAL_ADVERTISE_PEER_URLS} \
        --initial-cluster=${ETCD_INITIAL_CLUSTER} \
        --initial-cluster-token=${ETCD_INITIAL_CLUSTER_TOKEN} \
        --initial-cluster-state=new \
        --cert-file=/opt/etcd/ssl/server.pem \
        --key-file=/opt/etcd/ssl/server-key.pem \
        --peer-cert-file=/opt/etcd/ssl/server.pem \
        --peer-key-file=/opt/etcd/ssl/server-key.pem \
        --trusted-ca-file=/opt/etcd/ssl/ca.pem \
        --peer-trusted-ca-file=/opt/etcd/ssl/ca.pem
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

#### 错误2

```
error &#8220;remote error: tls: bad certificate&#8221;, ServerName &#8220;&#8221;
```

解答：证书拒绝链接，证书问题重新生成即可

### 2.5 把刚才生成的证书拷贝到配置文件中的位置：

`cp ca.pem server.pem server-key.pem  /opt/etcd/ssl`

### 2.6启动并设置开启启动：

```bash
# systemctl start etcd
# systemctl enable etcd

```

### 2.7 `scp`拷贝到其他两个节点

```bash
scp -r /opt/etcd root@node01:/opt/
scp /var/lib/systemd/system/etcd.service root@node01:/var/lib/systemd/system/
```



```
scp -r /opt/etcd root@node02:/opt/
scp /var/lib/systemd/system/etcd.service root@node02:/var/lib/systemd/system/
```



### 2.7 部署完成后，检查etcd集群状态

```bash
# /opt/etcd/bin/etcdctl \
--ca-file=ca.pem --cert-file=server.pem --key-file=server-key.pem \
--endpoints="https://10.240.0.11:2379,https:// 10.240.0.21:2379,https:// 10.240.0.22:2379" \
cluster-health
member 18218cfabd4e0dea is healthy: got healthy result from https://192.168.31.63:2379
member 541c1c40994c939b is healthy: got healthy result from https://192.168.31.65:2379
member a342ea2798d20705 is healthy: got healthy result from https://192.168.31.66:2379
cluster is healthy

```

如果输出上面信息，就说明集群部署成功。如果有问题第一步先看日志：`/var/log/message`
或 `journalctl -u etcd`