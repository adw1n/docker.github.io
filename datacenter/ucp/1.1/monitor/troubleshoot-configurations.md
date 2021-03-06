---
description: Learn how to troubleshoot your Docker Universal Control Plane cluster.
keywords: ectd, key, value, store, ucp
redirect_from:
- /ucp/kv_store/
- /ucp/monitor/troubleshoot-configurations/
title: Troubleshoot cluster configurations
---

Docker UCP persists configuration data on an [etcd](https://coreos.com/etcd/)
key-value store that is replicated on all controller nodes of
the UCP cluster. This key-value store is for internal use only, and should not
be used by other applications.

This article shows how you can access the key-value store, for
troubleshooting configuration problems in your cluster.

## Using the REST API

In this example we use `curl` for making requests to the key-value
store REST API, and `jq` to process the responses.

You can install these tools on a Ubuntu distribution by running:

```bash
$ sudo apt-get update && apt-get install curl jq
```

1. Use a client bundle to authenticate your requests.
[Learn more](../access-ucp/cli-based-access.md).

2. Use the REST API to access the cluster configurations.

```none
# $DOCKER_HOST and $DOCKER_CERT_PATH are set when using the client bundle
$ export KV_URL="https://$(echo $DOCKER_HOST | cut -f3 -d/ | cut -f1 -d:):12379"

$ curl -s \
    --cert ${DOCKER_CERT_PATH}/cert.pem \
    --key ${DOCKER_CERT_PATH}/key.pem \
    --cacert ${DOCKER_CERT_PATH}/ca.pem \
    ${KV_URL}/v2/keys | jq "."
```

To learn more about the key-value store API, check the
[etcd official documentation](https://coreos.com/etcd/docs/latest/api.html).


## Using a CLI client

The containers running the key-value store, include `etcdctl`, a command line
client for etcd. You can run it using the `docker exec` command.

The examples below assume you are logged in with ssh into a UCP controller node.

### Check the health of the etcd cluster

```none
$ docker exec -it ucp-kv etcdctl \
        --endpoint https://127.0.0.1:2379 \
        --ca-file /etc/docker/ssl/ca.pem \
        --cert-file /etc/docker/ssl/cert.pem \
        --key-file /etc/docker/ssl/key.pem \
        cluster-health

member 16c9ae1872e8b1f0 is healthy: got healthy result from https://192.168.122.64:12379
member c5a24cfdb4263e72 is healthy: got healthy result from https://192.168.122.196:12379
member ca3c1bb18f1b30bf is healthy: got healthy result from https://192.168.122.223:12379
cluster is healthy
```

On failure the command exits with an error code, and no output.

### Show the current value of a key

```none
$ docker exec -it ucp-kv etcdctl \
        --endpoint https://127.0.0.1:2379 \
        --ca-file /etc/docker/ssl/ca.pem \
        --cert-file /etc/docker/ssl/cert.pem \
        --key-file /etc/docker/ssl/key.pem \
        ls /docker/nodes

/docker/nodes/192.168.122.196:12376
/docker/nodes/192.168.122.64:12376
/docker/nodes/192.168.122.223:12376
```


### List the current members of the cluster

```none
$ docker exec -it ucp-kv etcdctl \
        --endpoint https://127.0.0.1:2379 \
        --ca-file /etc/docker/ssl/ca.pem \
        --cert-file /etc/docker/ssl/cert.pem \
        --key-file /etc/docker/ssl/key.pem \
        member list

16c9ae1872e8b1f0: name=orca-kv-192.168.122.64 peerURLs=https://192.168.122.64:12380 clientURLs=https://192.168.122.64:12379
c5a24cfdb4263e72: name=orca-kv-192.168.122.196 peerURLs=https://192.168.122.196:12380 clientURLs=https://192.168.122.196:12379
ca3c1bb18f1b30bf: name=orca-kv-192.168.122.223 peerURLs=https://192.168.122.223:12380 clientURLs=https://192.168.122.223:12379
```

### Remove a failed member

As long as your cluster is still functional and has not lost quorum
(more than (n/2)-1 nodes failed) you can use the following command to
remove the failed members.

```none
$ docker exec -it ucp-kv etcdctl \
        --endpoint https://127.0.0.1:2379 \
        --ca-file /etc/docker/ssl/ca.pem \
        --cert-file /etc/docker/ssl/cert.pem \
        --key-file /etc/docker/ssl/key.pem \
        member remove c5a24cfdb4263e72

Removed member c5a24cfdb4263e72 from cluster
```

If your cluster has lost too many members, etcd refuses to remove
members using this tool. Instead you must use the UCP backup and restore
functionality to reset your cluster to a single controller node cluster.
[Learn more about backups and disaster recovery](../high-availability/backups-and-disaster-recovery.md).


## Where to go next

* [Monitor your cluster](monitor-ucp.md)
* [Get support](../support.md)