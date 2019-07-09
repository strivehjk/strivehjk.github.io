---
title: kube-apiserver启动失败之etcd配置
date: 2019-01-06 22:17:00
category: kubernetes
tags:
	- etcd
	- k8s
description: kube-apiserver因为连接etcd失败导致服务启动失败，这和etcd的配置有关
---

在配置了认证之后，apiserver服务启动时有时候会访问etcd集群失败。
原因和etcd服务的配置有关。在其配置文件中，有几个访问URL相关的参数，
下面是本地能正常启动服务的配置， 要注意其http  和  http_s_的区别。

```
# Human_readable name for this member.
#
# default: "default"
#
# ETCD_NAME="default"

# Path to the data directory.
#
# default: "${name}.etcd"
# distribution default: "/var/lib/etcd"
#
# ETCD_DATA_DIR="/var/lib/etcd"

# Path to the dedicated wal directory.
# If this flag is set, etcd will write the WAL files
# to the walDir rather than the dataDir.
#
# default: ""
#
# ETCD_WAL_DIR=""

# Number of committed transactions to trigger a snapshot to disk.
#
# default: 10000
#
# ETCD_SNAPSHOT_COUNT=10000

# Time (in milliseconds) of a heartbeat interval.
#
# default: 100
#
# ETCD_HEARTBEAT_INTERVAL=100

# Time (in milliseconds) for an election to timeout.
#
# default: 1000
#
# ETCD_ELECTION_TIMEOUT=1000

# List of URLs to listen on for peer traffic.
#
# default: "http://localhost:2380,http://localhost:7001"
#
# ETCD_LISTEN_PEER_URLS="http://localhost:2380,http://localhost:7001"
ETCD_LISTEN_PEER_URLS="https://192.168.1.3:2380"

# List of URLs to listen on for client traffic.
#
# default: "http://localhost:2379,http://localhost:4001"
#
# ETCD_LISTEN_CLIENT_URLS="http://localhost:2379,http://localhost:4001"
ETCD_LISTEN_CLIENT_URLS="http://192.168.1.3:2379,http://127.0.0.1:2379"

# Maximum number of snapshot files to retain (0 is unlimited)
#
# default: 5
#
# ETCD_MAX_SNAPSHOTS=5

# Maximum number of wal files to retain (0 is unlimited)
#
# default: 5
#
# ETCD_MAX_WALS=5

# Comma_separated white list of origins for CORS (cross_origin resource sharing).
#
# default: none
#
# ETCD_CORS=

# List of this member's peer URLs to advertise to the rest of the cluster.
# These addresses are used for communicating etcd data around the cluster.
# At least one must be routable to all cluster members.
#
# default: "http://localhost:2380,http://localhost:7001"
#
# ETCD_INITIAL_ADVERTISE_PEER_URLS="http://localhost:2380,http://localhost:7001"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.1.3:2380"

# Initial cluster configuration for bootstrapping.
#
# default: "default=http://localhost:2380,default=http://localhost:7001"
# distribution default: "default=http://localhost:2380,default=http://localhost:7001"
#
# ETCD_INITIAL_CLUSTER="default=http://localhost:2380,default=http://localhost:7001"
# ETCD_INITIAL_CLUSTER="default=https://192.168.1.3:2380"

# Initial cluster state ("new" or "existing").
# Set to new for all members present during initial static or DNS bootstrapping.
# If this option is set to existing, etcd will attempt to join the existing cluster.
# If the wrong value is set, etcd will attempt to start but fail safely.
#
# default: "new"
#
# ETCD_INITIAL_CLUSTER_STATE="new"

# Initial cluster token for the etcd cluster during bootstrap.
#
# default: "etcd_cluster"
#
# ETCD_INITIAL_CLUSTER_TOKEN="etcd_cluster"

# List of this member's client URLs to advertise to the rest of the cluster.
#
# default: "http://localhost:2379,http://localhost:4001"
#
# ETCD_ADVERTISE_CLIENT_URLS="http://localhost:2379,http://localhost:4001"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.1.3:2379"

# Discovery URL used to bootstrap the cluster.
#
# default: none
#
# ETCD_DISCOVERY=

# DNS srv domain used to bootstrap the cluster.
#
# default: none
#
# ETCD_DISCOVERY_SRV=

# Expected behavior ("exit" or "proxy") when discovery services fails.
#
# default: "proxy"
#
# ETCD_DISCOVERY_FALLBACK="proxy"

# HTTP proxy to use for traffic to discovery service.
#
# default: none
#
# ETCD_DISCOVERY_PROXY=

# Proxy mode setting ("off", "readonly" or "on").
#
# default: "off"
#
# ETCD_PROXY="off"

# Time (in milliseconds) an endpoint will be held
# in a failed state before being reconsidered for proxied requests.
#
# default: 5000
#
# ETCD_PROXY_FAILURE_WAIT=5000

# Time (in milliseconds) of the endpoints refresh interval.
#
# default: 30000
#
# ETCD_PROXY_REFRESH_INTERVAL=30000

# Time (in milliseconds) for a dial to timeout or 0 to disable the timeout.
#
# default: 1000
#
# ETCD_PROXY_DIAL_TIMEOUT=1000

# Time (in milliseconds) for a write to timeout or 0 to disable the timeout.
#
# default: 5000
#
# ETCD_PROXY_WRITE_TIMEOUT=5000

# Time (in milliseconds) for a read to timeout or 0 to disable the timeout.
# Don't change this value if you use watches because they are using long polling requests.
#
# default: 0
#
# ETCD_PROXY_READ_TIMEOUT=0

# Path to the client server TLS CA file.
#
# default: none
#
# ETCD_CA_FILE=
ETCD_CA_FILE=/etc/kubernetes/pki/apiserver/ca.pem

# Path to the client server TLS cert file.
#
# default: none
#
# ETCD_CERT_FILE=
ETCD_CERT_FILE="/etc/kubernetes/pki/apiserver/etcd.pem"
#ETCD_CERT_FILE="--cert-file=/etc/kubernetes/pki/apiserver/etcd.pem"

# Path to the client server TLS key file.
#
# default: none
#
# ETCD_KEY_FILE=
ETCD_KEY_FILE="/etc/kubernetes/pki/apiserver/etcd-key.pem"
#ETCD_KEY_FILE="--key-file=/etc/kubernetes/pki/apiserver/etcd-key.pem"

# Enable client cert authentication.
#
# default: false
#
# ETCD_CLIENT_CERT_AUTH=false
ETCD_CLIENT_CERT_AUTH=true

# Path to the client server TLS trusted CA key file.
#
# default: none
#
# ETCD_TRUSTED_CA_FILE=
ETCD_TRUSTED_CA_FILE="/etc/kubernetes/pki/apiserver/ca.pem"
#ETCD_TRUSTED_CA_FILE="--trusted-ca-file=/etc/kubernetes/pki/apiserver/ca.pem"

# [DEPRECATED] Path to the peer server TLS CA file.
#
# default: none
#
# ETCD_PEER_CA_FILE=

# Path to the peer server TLS cert file.
#
# default: none
#
# ETCD_PEER_CERT_FILE=
#ETCD_PEER_CERT_FILE="--peer-cert-file=/etc/kubernetes/pki/apiserver/etcd.pem"
ETCD_PEER_CERT_FILE="/etc/kubernetes/pki/apiserver/etcd.pem"

# Path to the peer server TLS key file.
#
# default: none
#
# ETCD_PEER_KEY_FILE=
ETCD_PEER_KEY_FILE="/etc/kubernetes/pki/apiserver/etcd-key.pem"
#ETCD_PEER_KEY_FILE="--peer-key-file=/etc/kubernetes/pki/apiserver/etcd-key.pem"

# Enable peer client cert authentication.
#
# default: false
#
# ETCD_PEER_CLIENT_CERT_AUTH=false

# Path to the peer server TLS trusted CA file.
#
# default: none
#
# ETCD_PEER_TRUSTED_CA_FILE=
ETCD_PEER_TRUSTED_CA_FILE="/etc/kubernetes/pki/apiserver/ca.pem"
#ETCD_PEER_TRUSTED_CA_FILE="--peer-trusted-ca-file=/etc/kubernetes/pki/apiserver/ca.pem"

# Drop the default log level to DEBUG for all subpackages.
#
# default: false (INFO for all packages)
#
# ETCD_DEBUG=false

# Set individual etcd subpackages to specific log levels.
# An example being etcdserver=WARNING,security=DEBUG
#
# default: none (INFO for all packages)
#
# ETCD_LOG_PACKAGE_LEVELS=

# Force to create a new one_member cluster.
# It commits configuration changes in force to remove all existing members in the cluster and add itself.
# It needs to be set to restore a backup.
#
# default: false
#
# ETCD_FORCE_NEW_CLUSTER=false

# vim:ft=sh:
```
