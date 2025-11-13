# Lab 8: Algonquin Pet Store (On Steroids!)

## Demo Video

## Reflection Questions: Improving and Extending the Deployment

1. Make MongoDB capable of high availability (replication) and persistent storage:
   - PersistentVolumeClaim (PVC)
   - MongoDB Replica Set

Persistent storage in K8s orchestration starts with `PersistentVolumes`. A PV is a piece of storage provisioned in the K8s cluster that has a lifecycle independent of the Pod that would use the PV. Therefore, if a Pod is deleted or restarted, the PV remains intact; thus, storage may persist when it needs to.
`PersistentVolumeClaims` consume PV resources in a similar way that Pods consume node resources. While Pods request dedicated compute resources like CPU/memory, PVCs request dedicated storage size and access modes (ReadWriteOnce, ReadOnlyMany, etc.) ([ref](https://kubernetes.io/docs/concepts/storage/persistent-volumes/))

Replica sets in MongoDB are a group of `mongod` (daemon/engine for MongoDB) processes that maintain the same data set, thus providing high availability within production deployments by increasing redundancy. Since multiple copies of the same data are available across servers, replica sets provide fault tolerance to the system. ([ref](https://www.mongodb.com/docs/manual/replication/)) 

Within our K8s manifest, this is accomplished by having more than one pod replica in our MongoDB StatefulSet. By increasing to `replica: 3`, we scale out to 3 MongoDB pods that form the replica set. Furthermore, by adding `volumeClaimTemplates` to the StatefulSet manifest, each of those 3 pods have their own PVC to reinforce fault tolerance of data during pod restarts. Finally, the headless MongoDB service enables predictable & consistent DNS names like mongodb-0, mongodb-1, mongodb-2 to easily enable replica set members to reconnect to each other even across pod restarts.

One thing to note is that at the moment, the configuration above does not seem to make a true replica set for MongoDB; while the PVCs protect against data loss on pod restart/deletion, they do not yet replicate or synchronize data between the MongoDB pods. From some quick research, we could enable the replica set by adding an `initContainer` ([ref](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)) to the StatefulSet that runs the command `rs.initiate()` ([ref](https://www.mongodb.com/docs/manual/reference/method/rs.initiate/)) when the first pod is ready but before the service containers start running & storing data. Alternatively, we could use the Custom Resource Definitions (CRDs) provided by MongoDeb Kubernetes Operator so we can use the object `MongoDB` and create a replica set in our manifest that way.
([ref](https://www.mongodb.com/docs/kubernetes/current/tutorial/deploy-replica-set/))

```yaml
apiVersion: mongodb.com/v1
kind: MongoDB
metadata:
  name: mongodb
spec:
  members: 3 # akin to replica: 3
  version: "4.2"
  opsManager:
    configMapRef:
           # Must match metadata.name in ConfigMap file
      name: <configMap.metadata.name>
  credentials: <mycredentials>
  type: ReplicaSet
  persistent: true
```

2. Enable RabbitMQ to store messages persistently:
    - Mount a persistent volume at /var/lib/rabbitmq.
    - volumeClaimTemplates block to create per-pod PVCs
    - Hint: RabbitMQ stores its data in /var/lib/rabbitmq. Use PersistentVolumeClaim templates similar to MongoDB’s.

In order to ensure that the RabbitMQ pods persist data across pod restarts or deletions, we must add a `volumeMount` that tells RabbitMQ to store runtime data (queues & its messages) at the provided path `/var/lib/rabbitmq`, its default data path ([ref](https://kubernetes.io/docs/concepts/storage/volumes/)). Alongside that `volumeMount`, the `volumeClaimTemplates` tells K8s to create a PVC for each pod in the RabbitMQ StatefulSet. Therefore, each pod's PVC will bind to a PV in the cluster that is retained across pod restarts/failures by automatically reattaching itself to the pod when it comes alive again.

To summarize, when a RabbitMQ pod writes data to our defined `volumeMount` path `/var/lib/rabbitmq`, it actually writes to its PVC we define in `volumeClaimTemplates`. Each PVC persists across pod restarts/failures and reattaches to the pod so RabbitMQ sees its preserved data.

3. Investigate what Azure managed services could replace self-hosted MongoDB and RabbitMQ.
    - Give its name and purpose.
    - Explain why it’s a good fit (e.g., scaling, backups, availability).

Azure Cosmos DB and Azure Service Bus are the managed backing service that could replace MongoDB and RabbitMQ respectively.

Azure Cosmos DB is a fully managed NoSQL database with extremely low response times, high scalability potential, and guaranteed 99.999% availability, security, and business continuity (SLA-backed) at any scale. In addition, Cosmos DB decreases administrative overhead with automatic management, updates, and patching. Finally, elastic autoscaling and serverless options enable the database to match capacity to demand. ([ref](https://learn.microsoft.com/en-us/azure/cosmos-db/introduction))

Since Cosmos DB is fully managed by Azure, it would remove all the operational overhead of scaling replica sets or StatefulSet pods and database updates/patches. In addition, the near-100% guaranteed availability across global Azure regions enforces far stronger fault tolerance than self-managed MongoDB ever could. Finally, Cosmos DB can also automatically replicate data across regions to further ensure zero business downtime. In the APS system, it would complement the concurrency performance of makeline service's Go-based goroutines with its low latency and high thoroughput.

Azure Service Bus is a fully managed enterprise message broker with message queues and publish-subscribe topics. It is used to decouple services from each other, load-balance traffic across instances, and ensure delivery for data transfers that require high reliability.

As another fully managed service by Azure, Azure Service Bus would negative the orchestration management of ensuring that messages and queues are synced across all pods in a K8s node. Furthermore, Service Bus provides the additional benefit of automatically load balancing messages to multiple consumers reading from one queue; in APS, this would streamline reads to the makeline service from the message broker, which complements the highly concurrent potential of the Go service.