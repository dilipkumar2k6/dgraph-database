# Understanding Dgraph cluster
- Dgraph is a truly distributed graph database - not a master-slave replication of universal dataset. 
- It shards by predicate and replicates predicates across the cluster
- queries can be run on any node and joins are handled over the distributed data.
-  A query is resolved locally for predicates the node stores, and via distributed joins for predicates stored on other nodes.
- For effectively running a Dgraph cluster, it’s important to understand how sharding, replication and rebalancing works.
# Sharding
- Dgraph colocates data per predicate (* P *, in RDF terminology)
- thus the smallest unit of data is one predicate. 
- To shard the graph, one or many predicates are assigned to a group. 
- Each Alpha node in the cluster serves a single group. 
- Dgraph Zero assigns a group to each Alpha node.
# Shard rebalancing
- Dgraph Zero tries to rebalance the cluster based on the disk usage in each group. 
- If Zero detects an imbalance, it would try to move a predicate along with its indices to a group that has minimum disk usage. 
- This can make the predicate temporarily read-only. 
- Queries for the predicate will still be serviced, but any mutations for the predicate will be rejected and should be retried after the move is finished.
- Zero would continuously try to keep the amount of data on each server even, typically running this check on a 10-min frequency. 
- Thus, each additional Dgraph Alpha instance would allow Zero to further split the predicates from groups and move them to the new node.
# Consistent Replication
- If --replicas flag is set to something greater than one, Zero would assign the same group to multiple nodes. 
- These nodes would then form a Raft group aka quorum. 
- Every write would be consistently replicated to the quorum. 
- To achieve consensus, its important that the size of quorum be an odd number. 
- Therefore, we recommend setting --replicas to 1, 3 or 5 (not 2 or 4). 
- This allows 0, 1, or 2 nodes serving the same group to be down, respectively without affecting the overall health of that group.
# Ports Usage
Dgraph cluster nodes use different ports to communicate over gRPC and HTTP. 
# Types of Port
- gRPC-internal: Port that is used between the cluster nodes for internal communication and message exchange.
- gRPC-external: Port that is used by Dgraph clients, Dgraph Live Loader , and Dgraph Bulk loader to access APIs over gRPC.
- http-external: Port that is used by clients to access APIs over HTTP and other monitoring & administrative tasks.
# HA Cluster Setup
- In a high-availability setup, we need to run 3 or 5 replicas for Zero, and similarly, 3 or 5 replicas for Alpha.
- Note If number of replicas is 2K + 1, up to K servers can be down without any impact on reads or writes. 
- Avoid keeping replicas to 2K (even number). If K servers go down, this would block reads and writes, due to lack of consensus.
## Dgraph Zero 
- Run three Zero instances, assigning a unique ID(Integer) to each via --idx flag, and passing the address of any healthy Zero instance via --peer flag.
- To run three replicas for the alphas, set --replicas=3. Every time a new Dgraph Alpha is added, Zero would check the existing groups and assign them to one, which doesn’t have three replicas.

## Dgraph Alpha
- Run as many Dgraph Alphas as you want. 
- You can manually set --idx flag
- or you can leave that flag empty, and Zero would auto-assign an id to the Alpha. 
- This id would get persisted in the write-ahead log, so be careful not to delete it.
- The new Alphas will automatically detect each other by communicating with Dgraph zero and establish connections to each other.
- Typically, Zero would first attempt to replicate a group, by assigning a new Dgraph alpha to run the same group as assigned to another. 
- Once the group has been replicated as per the --replicas flag, Zero would create a new group.
- Over time, the data would be evenly split across all the groups. 
- So, it’s important to ensure that the number of Dgraph alphas is a multiple of the replication setting. 
- For e.g., if you set --replicas=3 in Zero, then run three Dgraph alphas for no sharding, but 3x replication. 
- Run six Dgraph alphas, for sharding the data into two groups, with 3x replication.
# HA Cluster Setup Using Kubernetes
- This setup allows you to run 3 Dgraph Alphas and 3 Dgraph Zeros. 
- We start Zero with --replicas 3 flag, so all data would be replicated on 3 Alphas and form 1 alpha group.
-  Ideally you should have at least three worker nodes as part of your Kubernetes cluster so that each Dgraph Alpha runs on a separate node.
https://github.com/dgraph-io/dgraph/blob/master/contrib/config/kubernetes/dgraph-ha/dgraph-ha.yaml

# Kubernetes Storage
- The Kubernetes configurations in the previous sections were configured to run Dgraph with any storage type (storage-class: anything). 
- On the common cloud environments like AWS, GCP, and Azure, the default storage type are slow disks like hard disks or low IOPS SSDs. We highly recommend using faster disks for ideal performance when running Dgraph.
## Local storage
- The AWS storage-optimized i-class instances provide locally attached NVMe-based SSD storage which provide consistent very high IOPS. The Dgraph team uses i3.large instances on AWS to test Dgraph.
- You can create a Kubernetes StorageClass object to provision a specific type of storage volume which you can then attach to your Dgraph pods. 
- You can set up your cluster with local SSDs by using Local Persistent Volumes. 
- This Kubernetes feature is in beta at the time of this writing (Kubernetes v1.13.1). 
- You can first set up an EC2 instance with locally attached storage.
-  Once it is formatted and mounted properly, then you can create a StorageClass to access it.:
## Removing a Dgraph Pod
- In the event that you need to completely remove a pod (e.g., its disk got corrupted and data cannot be recovered), you can use the /removeNode API to remove the node from the cluster. 
- With a Kubernetes StatefulSet, you’ll need to remove the node in this order:
1. Call /removeNode to remove the Dgraph instance from the cluster (see More about Dgraph Zero). 
2. The removed instance will immediately stop running. 
3. Any further attempts to join the cluster will fail for that instance since it has been removed.
4. Remove the PersistentVolumeClaim associated with the pod to delete its data.
5. This prepares the pod to join with a clean state.
6. Restart the pod. This will create a new PersistentVolumeClaim to create new data directories.

- When an Alpha pod restarts in a replicated cluster, it will join as a new member of the cluster, be assigned a group and an unused index from Zero, and receive the latest snapshot from the Alpha leader of the group.
- When a Zero pod restarts, it must join the existing group with an unused index ID. The index ID is set with the --idx flag. This may require the StatefulSet configuration to be updated.

# More about Dgraph Alpha
On its HTTP port, a Dgraph Alpha exposes a number of admin endpoints.
- /health
- /admin/shutdown
- /admin/export
# More about Dgraph Zero
- Dgraph Zero controls the Dgraph cluster. 
- It automatically moves data between different Dgraph Alpha instances based on the size of the data served by each Alpha instance.
- It is mandatory to run at least one dgraph zero node before running any dgraph alpha. 
- Options present for dgraph zero can be seen by running dgraph zero --help.
- Zero stores information about the cluster.
- --replicas is the option that controls the replication factor. (i.e. number of replicas per data shard, including the original shard)
- When a new Alpha joins the cluster, it is assigned a group based on the replication factor. If the replication factor is 1 then each Alpha node will serve different group. If replication factor is 2 and you launch 4 Alphas, then first two Alphas would serve group 1 and next two machines would serve group 2.
- Zero also monitors the space occupied by predicates in each group and moves them around to rebalance the cluster.
Like Alpha, Zero also exposes HTTP on 6080 (+ any --port_offset). You can query (GET request) it to see useful information, like the following:

- /state Information about the nodes that are part of the cluster. Also contains information about size of predicates and groups they belong to.
- /assign?what=uids&num=100 This would allocate num uids and return a JSON map containing startId and endId, both inclusive. This id range can be safely assigned externally to new nodes during data ingestion.
- /assign?what=timestamps&num=100 This would request timestamps from Zero. This is useful to fast forward Zero state when starting from a postings directory, which already has commits higher than Zero’s leased timestamp.
- /removeNode?id=3&group=2 If a replica goes down and can’t be recovered, you can remove it and add a new node to the quorum. This endpoint can be used to remove a dead Zero or Dgraph Alpha node. To remove dead Zero nodes, pass group=0 and the id of the Zero node.
- /moveTablet?tablet=name&group=2 This endpoint can be used to move a tablet to a group. Zero already does shard rebalancing every 8 mins, this endpoint can be used to force move a tablet.
- POST /enterpriseLicense Use endpoint to apply an enterprise license to the cluster by supplying it as part of the body.
# More about /state endpoint
```
{
  "counter": "15",
  "groups": {
    "1": {
      "members": {
        "1": {
          "id": "1",
          "groupId": 1,
          "addr": "alpha1:7080",
          "leader": true,
          "lastUpdate": "1576112366"
        },
        "2": {
          "id": "2",
          "groupId": 1,
          "addr": "alpha2:7080"
        },
        "3": {
          "id": "3",
          "groupId": 1,
          "addr": "alpha3:7080"
        }
      },
      "tablets": {
        "counter.val": {
          "groupId": 1,
          "predicate": "counter.val"
        },
        "dgraph.type": {
          "groupId": 1,
          "predicate": "dgraph.type"
        }
      },
      "checksum": "1021598189643258447"
    }
  },
  "zeros": {
    "1": {
      "id": "1",
      "addr": "zero1:5080",
      "leader": true
    },
    "2": {
      "id": "2",
      "addr": "zero2:5080"
    },
    "3": {
      "id": "3",
      "addr": "zero3:5080"
    }
  },
  "maxLeaseId": "10000",
  "maxTxnTs": "10000",
  "cid": "3602537a-ee49-43cb-9792-c766eea683dc",
  "license": {
    "maxNodes": "18446744073709551615",
    "expiryTs": "1578704367",
    "enabled": true
  }
}
```
# TLS configuration
- Connections between client and server can be secured with TLS. 
- Password protected private keys are not supported.
## Self-signed certificates
The dgraph cert program creates and manages self-signed certificates using a generated Dgraph Root CA. The cert command simplifies certificate management for you.
```
# To see the available flags.
$ dgraph cert --help

# Create Dgraph Root CA, used to sign all other certificates.
$ dgraph cert

# Create node certificate (needed for Dgraph Live Loader using TLS)
$ dgraph cert -n live

# Create client certificate
$ dgraph cert -c dgraphuser

# Combine all in one command
$ dgraph cert -n live -c dgraphuser

# List all your certificates and keys
$ dgraph cert ls
```
## File naming conventions
- To enable TLS you must specify the directory path to find certificates and keys. 
- The default location where the cert command stores certificates (and keys) is tls under the Dgraph working directory; where the data files are found. 
- The default dir path can be overridden using the --dir option.
### ca.crt
Dgraph Root CA certificate. Verify all certificates
### ca.key	
Dgraph CA private key. Validate CA certificate
### node.crt	
Dgraph node certificate. Shared by all nodes for accepting TLS connections
### node.key	
Dgraph node private key. Validate node certificate
### client.name.crt	
Dgraph client certificate. Authenticate a client name
### client.name.key	
Dgraph client private key. Validate name client certificate

The Root CA certificate is used for verifying node and client certificates, if changed you must regenerate all certificates.

## Using Ratel UI with Client authentication
## Using Curl with Client authentication
```
curl --cacert ./tls/ca.crt https://localhost:8080/admin/export

```
# Monitoring
Dgraph exposes metrics via the /debug/vars endpoint in json format and the /debug/prometheus_metrics endpoint in Prometheus’s text-based format.
















