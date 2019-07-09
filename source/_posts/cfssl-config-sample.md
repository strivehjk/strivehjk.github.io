---
title: cfssl-config-sample
date: 2019-01-01 22:07:12
tags:
description: CFSSL CA和config文件 JSON格式示例和字段含意说明
---

## kubernetes认证文件生成工具 CFSSL 

在生成认证文件的时候，往往会用到， 网上很多有示例文件，一直不明白其中的CN，L，OU，什么的是什么意思。
在`https://k8smeetup.github.io/docs/concepts/cluster-administration/certificates/`上有如下介绍：
```
C = <country>
ST = <state>
L = <city>
O = <organization>
OU = <organization unit>
CN = <MASTER_IP>
```

## 生成根证书，用以下命令：

> $ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
> #生成完成后会有以下文件（我们最终想要的就是ca-key.pem和ca.pem，一个秘钥，一个证书）
> $ ls
> ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem

上述命令中用到的ca-csr.json是：
```
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "XS",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

## CSFFL 工具在生成认证文件的时候，用到以下选项(cfssl gencert)：
```
Usage of gencert:
    Generate a new key and cert from CSR:
        cfssl gencert -initca CSRJSON
        cfssl gencert -ca cert -ca-key key [-config config] [-profile profile] [-hostname hostname] CSRJSON
        cfssl gencert -remote remote_host [-config config] [-profile profile] [-label label] [-hostname hostname] CSRJSON

    Re-generate a CA cert with the CA key and CSR:
        cfssl gencert -initca -ca-key key CSRJSON

    Re-generate a CA cert with the CA key and certificate:
        cfssl gencert -renewca -ca cert -ca-key key

Arguments:
        CSRJSON:    JSON file containing the request, use '-' for reading JSON from stdin

Flags:
  -initca=false: initialise new CA
  -remote="": remote CFSSL server
  -ca="": CA used to sign the new certificate -- accepts '[file:]fname' or 'env:varname'
  -ca-key="": CA private key -- accepts '[file:]fname' or 'env:varname'
  -config="": path to configuration file
  -cn="": certificate common name (CN)
  -hostname="": Hostname for the cert, could be a comma-separated hostname list
  -profile="": signing profile to use
  -label="": key label to use in remote CFSSL server
  -loglevel=1: Log level (0 = DEBUG, 5 = FATAL)
```
### CFSSL config 文件
```
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}

```

## 生成证书

```
$ cfssl gencert \
        -ca=/path/to/ca.pem \
        -ca-key=/path/to/ca-key.pem \
        -config=/path/to/ca-config.json \
        -profile=kubernetes admin-csr.json | cfssljson -bare admin
```
 我们最终要的是admin-key.pem和admin.pem
