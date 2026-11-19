# CST8915 Lab 7 – Kubernetes Basics and RabbitMQ Configuration Analysis

**Student:** Hanan Ali

**Student ID:** 041153451

**Course:** CST8915 Full-stack Cloud-native Development

**Semester:** Fall 2026

## Demo Video

[Watch Demo Video](https://youtu.be/f3NqN6QIsHw)

## RabbitMQ Configuration Analysis

### Is RabbitMQ Stateless or Stateful?
RabbitMQ is a **stateful** application. Its purpose is to accept messages from producers and hold them until a consumer processes them, so the queued messages, queue definitions, exchanges, and user accounts are all state that the broker must keep. In the Algonquin Pet Store, every order placed in the Store Front is published by the Order Service to `order_queue`, where it waits until a consumer reads it.

The other three components are stateless. The Order Service, Product Service, and Store Front do not keep data between requests, so any replacement Pod can serve traffic immediately. This is why a Deployment suits them but not RabbitMQ: a Deployment treats its Pods as interchangeable and disposable, while Kubernetes provides StatefulSets specifically for applications that need persistent storage or a stable network identity [1].

### Implications of Running RabbitMQ Without Persistent Storage
The RabbitMQ Deployment in `algonquin-pet-store-all-in-one.yaml` defines no `volumes` or `volumeMounts`, so the broker writes all of its data to the container's own writable filesystem. Kubernetes documents that on-disk files in a container are ephemeral: when a container crashes or is stopped, the files created or modified during its lifetime are lost [2]. This has several implications:

- **Silent message loss:** Orders waiting in `order_queue` exist only inside the running container. If that container is replaced, the orders are lost, even though the Store Front already showed the customer a "placed successfully" message.
- **Durability settings do not help:** `order_queue` is declared as durable (shown with a "D" in the management UI). RabbitMQ stores the metadata of durable queues on disk and writes persistent messages to disk [3], but here that disk is the container's temporary filesystem. The RabbitMQ team states that a node's data directory must use durable storage, with a Persistent Volume attached to each RabbitMQ Pod, for data to survive Pod restarts [4].
- **Runtime configuration is lost:** Queues, exchanges, bindings, and any users or permissions created after startup disappear. Only the default user is recreated, because it comes from the `RABBITMQ_DEFAULT_USER` and `RABBITMQ_DEFAULT_PASS` environment variables.
- **Single point of failure:** With `replicas: 1` and no shared storage, there is no second broker to take over. While the replacement Pod starts, the broker is unavailable; in the demo, the management dashboard returned `502 Bad Gateway` until the new Pod was ready.

### What Happens When the RabbitMQ Pod Is Deleted or Restarted
The demo video reproduces this problem:

1. Two orders were placed through the Store Front, and the management UI showed `order_queue` holding **2 messages**.
2. The RabbitMQ Pod (`rabbitmq-5d4b559c94-rb2ll`) was deleted with `kubectl delete pod`.
3. The Deployment immediately created a replacement Pod (`rabbitmq-5d4b559c94-mn54h`) to maintain `replicas: 1`. While it started, the management dashboard returned `502 Bad Gateway`.
4. Once the new Pod was running, the management UI showed **no queues**. The queue and both orders were gone.
5. A new order was then placed. The Order Service reconnected to the new broker and created a fresh `order_queue` containing only **1 message**; the earlier orders were never recovered.

Kubernetes self-healing restored the *service*, but not the *data*. The replacement Pod started from the clean `rabbitmq:3-management` image with an empty data directory, because nothing outside the original container held a copy of it. The management UI also showed the broker running under a new node name (`rabbit@rabbitmq-5d4b559c94-mn54h`), since a Deployment gives each new Pod a random name. RabbitMQ ties its stored data to the node name, which is one of the reasons the RabbitMQ team recommends a StatefulSet, with its stable identity, over a Deployment [4].

The same loss happens when the Pod is replaced for other reasons, such as node failure, eviction, or a rolling update. Even if only the container restarts inside the same Pod, files written to the container's filesystem are not preserved [2].

### Potential Solutions

### Does Azure Service Bus Solve These Issues?

## Setup Challenges and Lessons Learned

## References
1. Kubernetes Documentation – StatefulSets: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
2. Kubernetes Documentation – Volumes: https://kubernetes.io/docs/concepts/storage/volumes/
3. RabbitMQ Documentation – Queues (Durability): https://www.rabbitmq.com/docs/queues
4. RabbitMQ Blog – Deploying RabbitMQ to Kubernetes: What's Involved?: https://www.rabbitmq.com/blog/2020/08/10/deploying-rabbitmq-to-kubernetes-whats-involved