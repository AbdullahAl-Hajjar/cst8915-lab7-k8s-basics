# CST8915 Lab 7: Introduction to Kubernetes Basics

**Student Name**: Abdullah Al-Hajjar
**Student ID**: 040656012
**Course**: CST8915 Full-stack Cloud-native Development
**Semester**: Fall 2026

---

## Demo Video

🎥 [Watch Demo Video](YOUTUBE_LINK_HERE)

---

## Files

| File | Purpose |
| --- | --- |
| `my-pod.yaml` | A single NGINX Pod (Step 3). |
| `my-deployment.yaml` | An NGINX Deployment with 3 replicas, labelled `app: nginx` (Step 3). |
| `my-service.yaml` | A `LoadBalancer` Service that routes to the `app: nginx` Pods (Step 3). |
| `algonquin-pet-store-all-in-one.yaml` | Deployments and Services for RabbitMQ, Order Service, Product Service and Store Front (Step 4). |

## Cluster Setup

- **AKS cluster:** `Lab7Cluster` in resource group `CST8915-Lab7`, with one node (Kubernetes 1.36). The Free tier was used.
- **Region:** North Central US. My Azure for Students subscription only allows a few regions, and Canada is not one of them.
- **Node size:** `Standard_B2as_v2` (2 vCPU, 8 GB). D2as_v4 is not allowed on the subscription, and the B-series is the only family with available vCPU quota.

---

## Analysis: The RabbitMQ Configuration Problem

### What I observed

I deployed the pet store and placed three orders. The RabbitMQ management API showed them waiting in `order_queue`:

```
queue order_queue durable True messages 3 node rabbit@rabbitmq-5d4b559c94-sprpd
```

Then I deleted the RabbitMQ Pod with `kubectl delete pod -l app=rabbitmq`. The Deployment created a replacement Pod within seconds, but the queue and all three orders were gone:

```
GET /rabbitmq/api/queues/%2F/order_queue  ->  404 {"error":"Object Not Found"}
GET /rabbitmq/api/queues                  ->  []
node rabbit@rabbitmq-5d4b559c94-xq5n9
```

The Order Service logs still said `Sent order to queue` for each of the three orders. From the customer's point of view the orders were accepted, but they no longer exist anywhere. The queue only came back when the next order was placed, because the Order Service declares the queue again before it publishes.

### Is RabbitMQ stateless or stateful?

RabbitMQ is a **stateful** application. A message broker exists to hold data that has not been processed yet, so the messages themselves are its state. It also stores queue and exchange definitions, users, permissions and virtual hosts in its internal database (Mnesia/Khepri) under `/var/lib/rabbitmq`.

The Store Front and Product Service are different. Any copy of them can be replaced by a new one without losing anything, so they are stateless and are a good fit for a Deployment.

### Implications of running RabbitMQ without persistent storage

The manifest runs RabbitMQ as a plain Deployment with **no volume**. Inside the Pod I found that the data directory is on the node's temporary container filesystem:

```
$ kubectl exec deploy/rabbitmq -- ls /var/lib/rabbitmq/mnesia
rabbit@rabbitmq-5d4b559c94-xq5n9
...
$ df -h /var/lib/rabbitmq
/dev/root  123G  23G  101G  19% /var/lib/rabbitmq
```

That causes three problems:

1. **The data lives and dies with the container.** When the Pod is replaced, the new container starts with an empty filesystem.
2. **The node name changes on every Pod.** RabbitMQ names its data directory after the node (`rabbit@<hostname>`), and a Deployment gives each new Pod a random name. A new Pod would look for a different directory even if the old files were still there. That means adding a volume to the Deployment alone is not enough; the identity must be stable as well.
3. **Durable queues and persistent messages do not help.** The Order Service already does the right thing in code: `assertQueue(queue, { durable: true })` and `sendToQueue(..., { persistent: true })`. "Durable" and "persistent" only mean that RabbitMQ writes the data to disk. Here the disk itself is thrown away, so the guarantee is meaningless.

Other risks follow from this:
- **Lost orders:** any order not yet consumed is gone.
- **Silent failure:** the customer and the Order Service were both told the order succeeded.
- **Lost runtime settings:** any users or policies added at runtime would also vanish. Only `myuser` survives, because it is recreated from environment variables.
- **A single point of failure:** there is one replica, so the broker is unavailable while the new Pod starts.

### What happens when the RabbitMQ Pod is deleted or restarted

- **Pod deleted, evicted, or rescheduled** (node failure, node upgrade, `kubectl rollout restart`, or a scale event): the Deployment creates a new Pod with a new name and an empty filesystem. All queues and unconsumed messages are lost, as shown above.
- **Container restarted inside the same Pod** (for example after a crash or an out-of-memory kill at the 512Mi limit): the container's writable layer is also discarded on restart. The data does not survive that either.
- **During the gap:** while the replacement starts (about 15 seconds in my test), the Order Service cannot connect, and new orders fail with `ECONNREFUSED`. I saw this in Lab 6 when an order was placed before RabbitMQ was ready.

### Potential solutions

1. **Use a StatefulSet with a PersistentVolumeClaim.** A StatefulSet gives each Pod a stable name (`rabbitmq-0`) and its own PersistentVolumeClaim created from `volumeClaimTemplates`. The volume is reattached to the replacement Pod, so the node name and data directory stay the same. On AKS, the built-in `managed-csi` storage class provisions an Azure Managed Disk for the claim. A headless Service gives the Pod a stable DNS name. (Kubernetes, n.d.-a; Microsoft, n.d.-a)
2. **Fix the node name.** Set `RABBITMQ_NODENAME` (or the Pod `hostname`) so the data directory does not depend on a random Pod name.
3. **Use the RabbitMQ Cluster Operator.** Instead of hand-writing these manifests, install the official operator and create a `RabbitmqCluster` resource. The operator creates a StatefulSet with persistent storage, a stable node identity, and safe rolling upgrades. (RabbitMQ, n.d.-b)
4. **Make it highly available.** Run three replicas and use **quorum queues**, which replicate each message across a majority of nodes using the Raft protocol. Losing one Pod or disk then loses no data. (RabbitMQ, n.d.-c)
5. **Make the application safer.** Use **publisher confirms**, so the Order Service only tells the customer "Order received" after the broker has saved the message. Add retries with backoff, so short broker outages do not fail orders. (RabbitMQ, n.d.-a)
6. **Back up definitions.** Export the RabbitMQ definitions so users, queues and policies can be restored after a disaster.

### Does Azure Service Bus solve these issues?

**Mostly yes, for the data-loss problem.** Azure Service Bus is a fully managed message broker (PaaS). Azure stores and replicates the messages on its own durable infrastructure, outside the AKS cluster. Deleting, restarting or rescheduling Pods cannot delete the queue or its messages, and there is no broker Pod, disk, or StatefulSet to manage. It also adds features this lab would otherwise have to build: dead-letter queues, duplicate detection, scheduled messages, and availability-zone support in the Premium tier (Microsoft, n.d.-b).

It is **not a drop-in fix**, though:

- **Protocol:** Service Bus speaks **AMQP 1.0**, while the Order Service uses `amqplib`, an AMQP 0-9-1 client. The code would need to change to the `@azure/service-bus` SDK or another AMQP 1.0 client (Microsoft, n.d.-c).
- **Credentials:** the connection would need a Service Bus connection string or a managed identity, stored as a Kubernetes Secret rather than in plain YAML.
- **Cost and lock-in:** it is a paid Azure service, and the application becomes tied to Azure instead of running anywhere Kubernetes runs.
- **Publishing must still be safe:** if the Order Service crashes before the send completes, that order can still be lost. The application should only confirm the order after the send succeeds.

In short, Service Bus moves the stateful part (the queue) out of the cluster to a service built to keep it safe. That removes the persistence problem shown in this lab, at the cost of code changes and Azure lock-in. Staying with RabbitMQ is also valid, but it must run as a StatefulSet with persistent volumes, ideally through the Cluster Operator with quorum queues.

---

## Notes and Lessons Learned

- Two Azure for Students restrictions shaped the setup:
  - Canada regions and `D2as_v4` are blocked, so I used North Central US and `Standard_B2as_v2`.
  - The `Microsoft.ContainerService` resource provider had to be registered before AKS could be created.
- Kubernetes self-healing (recreating a deleted Pod) only restores the **Pod**, not its **data**. A green `Running` status does not mean nothing was lost.
- The `kubectl run`, `create` and `expose` commands are fast for experiments. The YAML files are what I would keep in Git, because they can be reviewed, versioned and re-applied.
- All Azure resources (resource group `CST8915-Lab7` and its `MC_` node resource group) were deleted after recording.

## References

Kubernetes. (n.d.-a). *StatefulSets*. https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/

Kubernetes. (n.d.-b). *Persistent volumes*. https://kubernetes.io/docs/concepts/storage/persistent-volumes/

Microsoft. (n.d.-a). *Create and use a volume with Azure Disks in Azure Kubernetes Service (AKS)*. https://learn.microsoft.com/en-us/azure/aks/azure-csi-disk-storage-provision

Microsoft. (n.d.-b). *What is Azure Service Bus?* https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview

Microsoft. (n.d.-c). *AMQP 1.0 in Azure Service Bus and Event Hubs protocol guide*. https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-amqp-protocol-guide

RabbitMQ. (n.d.-a). *Consumer acknowledgements and publisher confirms*. https://www.rabbitmq.com/docs/confirms

RabbitMQ. (n.d.-b). *RabbitMQ Cluster Kubernetes Operator*. https://www.rabbitmq.com/kubernetes/operator/operator-overview

RabbitMQ. (n.d.-c). *Quorum queues*. https://www.rabbitmq.com/docs/quorum-queues
