# [Portworx](http://portworx.com) for IKS

[Portworx](http://portworx.com) is the highly available persistent storage layer for Kubernetes
Documentation for Portworx can be found at [https://docs.portworx.com](https://docs.portworx.com)

# Installing Portworx in IKS

## Worker node pre-reqs
Ensure you have unmounted/unformatted raw block devices presented to 
the worker nodes.   Either make sure your "bring-your-own hardware" nodes have
available raw block devices.   Or follow the steps [here](https://github.com/akgunjal/block-volume-attacher)
to allocate remote block devices

## Provision a `Compose etcd` instance
-- 
Create and deploy an instance of [Compose for etcd](https://console.bluemix.net/catalog/services/compose-for-etcd)

## etcd

Use one OR the other of the two approaches below for providing Portworx with an 'etcd' instance

### Obtain the `etcd` username, password and endpoints

```
$ bx target bx target --cf
$ bx service list                        #  To find name of Compose etcd service
$ bx service show 'Compose for etcd-8n'  #  Use appropriate service name to retrieve the `dashboard` parameter for your corresponding `etcd-service`


$ ETCDCTL_API=3 etcdctl --endpoints=https://portal-ssl294-1.bmix-wdc-yp-a7a89461-abcc-45e5-84d7-cde68723e30d.588786498.composedb.com:15832,https://portal-ssl275-2.bmix-wdc-yp-a7a89461-abcc-45e5-84d7-cde68723e30d.588786498.composedb.com:15832 --user=root:SPATNQDYPNBYUIRR member list -w table
+------------------+---------+------------------------------------------+-------------------------+-------------------------+
|        ID        | STATUS  |                   NAME                   |       PEER ADDRS        |      CLIENT ADDRS       |
+------------------+---------+------------------------------------------+-------------------------+-------------------------+
|  418ef52006c5049 | started | etcd129.sl-us-wdc-1-memory.2.dblayer.com | http://10.96.146.3:2380 | http://10.96.146.3:2379 |
|  bc4aa29071d56be | started | etcd113.sl-us-wdc-1-memory.0.dblayer.com | http://10.96.146.4:2380 | http://10.96.146.4:2379 |
| a52c701de81e1e64 | started | etcd113.sl-us-wdc-1-memory.1.dblayer.com | http://10.96.146.2:2380 | http://10.96.146.2:2379 |
+------------------+---------+------------------------------------------+-------------------------+-------------------------+
```

Please make note of the `etcd-endpoints` as well as the `--user=root:<PASSWORD>` string 

### Use the Portworx internal `etcd` 

With Portworx 1.4, an [internal kvdb option is offered](https://docs.portworx.com/scheduler/kubernetes/install.html#internal-kvdb-beta)

## Deploy Portworx via Helm Chart

Make sure you have installed `helm` and run `helm init`

Follow these instructions to [Deploy Portworx via Helm](https://github.com/portworx/helm/blob/master/charts/portworx/README.md)

The following values must be defined, either through `helm install --set ...` or through `values.yaml`:
* clusterName      :   User defined
* etcd.credentials :   `root:<PASSWORD>` , where <PASSWORD> is taken from the above etcd URL
* etcdEndPoint     :   of the form `https://portal-ssl294-1.bmix-wdc-yp-a7a89461-abcc-45e5-84d7-cde68723e30d.588786498.composedb.com:15832`
                       where the actual URLs correspond to your `etcd` URL.

```
helm install --debug --name portworx-ds --no-hooks 


## Verify Portworx has been deployed
