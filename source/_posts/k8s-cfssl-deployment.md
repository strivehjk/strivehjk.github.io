---
title: k8s-cfssl-deployment
date: 2018-12-31 10:46:05
tags: 
- k8s
- CA
layout:
updated:
comments: true
categories:
permalink:
description: K8S 认证工具CFSSL的使用及kube服务的配置
---

Source ：http://blog.simlinux.com/archives/1953.html

更详细的介绍，请参照 ： 
`https://coreos.com/os/docs/latest/generate-self-signed-certificates.html`

## 容器相关证书类型
```
client certificate： 用于服务端认证客户端,例如etcdctl、etcd proxy、fleetctl、docker客户端
server certificate: 服务端使用，客户端以此验证服务端身份,例如docker服务端、kube-apiserver
peer certificate: 双向证书，用于etcd集群成员间通信
```
## 创建CA证书
### 生成默认CA配置
```
mkdir /opt/ssl
cd /opt/ssl
cfssl print-defaults config > ca-config.json
cfssl print-defaults csr > ca-csr.json
```
### 修改ca-config.json,分别配置针对三种不同证书类型的profile,其中有效期43800h为5年
```
{
    "signing": {
        "default": {
            "expiry": "43800h"
        },
        "profiles": {
            "server": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            },
            "client": {
                "expiry": "43800h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            },
            "peer": {
                "expiry": "43800h",
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
### 修改ca-csr.config
```
{
    "CN": "My own CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "US",
            "L": "CA",
            "O": "My Company Name",
            "ST": "San Francisco",
            "OU": "Org Unit 1",
            "OU": "Org Unit 2"
        }
    ]
}
```

### 生成CA证书和私钥
```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca - 
```
生成ca.pem、ca.csr、ca-key.pem(CA私钥,需妥善保管)

## 签发Server Certificate
cfssl print-defaults csr > server-csr.json
```
...
    "CN": "coreos1",
    "hosts": [
        "192.168.122.68",
        "ext.example.com",
        "coreos1.local",
        "coreos1"
    ],
...
```
### 生成服务端证书和私钥
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server server-csr.json | cfssljson -bare server
```
## 签发Client Certificate

```
cfssl print-defaults csr > admin-csr.json
```
```
...
    "CN": "client",
    "hosts": [""],
...
```
### 生成客户端证书和私钥
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client admin-csr.json | cfssljson -bare admin
```
## 签发peer certificate

```
cfssl print-defaults csr > kube-proxy-csr.json
```
```
...
    "CN": "member1",
    "hosts": [
        "192.168.122.101",
        "ext.example.com",
        "member1.local",
        "member1"
    ],
...
```
### 为节点member1生成证书和私钥:
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer kube-proxy-csr.json | cfssljson -bare kube-proxy
```
针对etcd服务,每个etcd节点上按照上述方法生成相应的证书和私钥

## 最后校验证书
校验生成的证书是否和配置相符
```
openssl x509 -in ca.pem -text -noout
openssl x509 -in server.pem -text -noout
openssl x509 -in client.pem -text -noout
```

## KUBERNETES服务配置

### 生成随机token文件

```
#生成随机token
$ head -c 16 /dev/urandom | od -An -t x | tr -d ' '
8afdf3c4eb7c74018452423c29433609

#按照固定格式写入token.csv，注意替换token内容
$ echo "8afdf3c4eb7c74018452423c29433609,kubelet-bootstrap,10001,\"system:kubelet-bootstrap\"" > /etc/kubernetes/ca/kubernetes/token.csv
```
在apiserver服务启动时添加配置项`KUBE_TOKEN_AUTH_FILE="--token-auth-file=/etc/kubernetes/pki/apiserver/token.csv"`

### master 节点配置 kubectl (下面 admin 即是生成的kubectl证书 )
```
#指定apiserver的地址和证书位置（ip自行修改）
$ kubectl config set-cluster kubernetes \
        --certificate-authority=/etc/kubernetes/ca/ca.pem \
        --embed-certs=true \
        --server=https://192.168.1.102:6443
#设置客户端认证参数，指定admin证书和秘钥
$ kubectl config set-credentials admin \
        --client-certificate=/etc/kubernetes/ca/admin/admin.pem \
        --embed-certs=true \
        --client-key=/etc/kubernetes/ca/admin/admin-key.pem
#关联用户和集群
$ kubectl config set-context kubernetes \
        --cluster=kubernetes --user=admin
#设置当前上下文
$ kubectl config use-context kubernetes

#设置结果就是一个配置文件，可以看看内容
$ cat ~/.kube/config
```
我们这里让kubelet使用引导token的方式认证，所以认证方式跟之前的组件不同，它的证书不是手动生成，而是由工作节点TLS BootStrap 向api-server请求，由主节点的controller-manager 自动签发。

#### 主节点创建角色绑定
引导token的方式要求客户端向api-server发起请求时告诉他你的用户名和token，并且这个用户是具有一个特定的角色：system:node-bootstrapper，
所以需要先将 bootstrap token 文件中的 kubelet-bootstrap 用户赋予这个特定角色，然后 kubelet 才有权限发起创建认证请求。 在主节点执行下面命令
```
#可以通过下面命令查询clusterrole列表
$ kubectl -n kube-system get clusterrole

#可以回顾一下token文件的内容
$ cat /etc/kubernetes/ca/kubernetes/token.csv
8afdf3c4eb7c74018452423c29433609,kubelet-bootstrap,10001,"system:kubelet-bootstrap"

#创建角色绑定（将用户kubelet-bootstrap与角色system:node-bootstrapper绑定）
$ kubectl create clusterrolebinding kubelet-bootstrap \
         --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
```
#### 工作节点创建bootstrap.kubeconfig
这个配置是用来完成bootstrap token认证的，保存了像用户，token等重要的认证信息，这个文件可以借助kubectl命令生成：（也可以自己写配置）
```
#设置集群参数(注意替换ip)
$ kubectl config set-cluster kubernetes \
        --certificate-authority=/etc/kubernetes/ca/ca.pem \
        --embed-certs=true \
        --server=https://192.168.1.102:6443 \
        --kubeconfig=bootstrap.kubeconfig
#设置客户端认证参数(注意替换token)
$ kubectl config set-credentials kubelet-bootstrap \
        --token=8afdf3c4eb7c74018452423c29433609 \
        --kubeconfig=bootstrap.kubeconfig
#设置上下文
$ kubectl config set-context default \
        --cluster=kubernetes \
        --user=kubelet-bootstrap \
        --kubeconfig=bootstrap.kubeconfig
#选择上下文
$ kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
```
kubelet.service服务添加下面的参数：         
```
KUBELET_POD_INFRA_CONTAINER_IMAGE="--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/imooc/pause-amd64:3.0"
KUBELET_EXPERIMENTAL_BOOTSTRAP_KUBECONFIG="--experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig"
KUBELET_CERT_DIR="--cert-dir=/etc/kubernetes/pki/apiserver"
KUBELET_HAIRPIN_MODE="--hairpin-mode hairpin-veth"
```
### kube-proxy 服务
    kube-proxy服务也需要生成证书。
    
    另外，需要生成kube-proxy.kubeconfig配置:
```
#设置集群参数（注意替换ip）
$ kubectl config set-cluster kubernetes \
        --certificate-authority=/etc/kubernetes/ca/ca.pem \
        --embed-certs=true \
        --server=https://192.168.1.102:6443 \
        --kubeconfig=kube-proxy.kubeconfig
#置客户端认证参数
$ kubectl config set-credentials kube-proxy \
        --client-certificate=/etc/kubernetes/ca/kube-proxy/kube-proxy.pem \
        --client-key=/etc/kubernetes/ca/kube-proxy/kube-proxy-key.pem \
        --embed-certs=true \
        --kubeconfig=kube-proxy.kubeconfig
#设置上下文参数
$ kubectl config set-context default \
        --cluster=kubernetes \
        --user=kube-proxy \
        --kubeconfig=kube-proxy.kubeconfig
#选择上下文
$ kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```
