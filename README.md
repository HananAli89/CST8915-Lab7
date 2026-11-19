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

### What Happens When the RabbitMQ Pod Is Deleted or Restarted

### Potential Solutions

### Does Azure Service Bus Solve These Issues?

## Setup Challenges and Lessons Learned

## References
1. Kubernetes Documentation – StatefulSets: https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/