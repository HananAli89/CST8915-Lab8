# CST8915 Lab 8 – Deploying the Algonquin Pet Store (On Steroids)

**Student:** Hanan Ali

**Student ID:** 041153451

**Course:** CST8915 Full-stack Cloud-native Development

**Semester:** Fall 2026

## Demo Video

[Watch Demo Video](https://youtu.be/nrsdiQBQfII)

## Task 1: Custom Docker Images

All eight application services were forked, built, and pushed to my Docker Hub account. Each image was built from my fork of the service repository, and the deployment files in this repository use my images:

| Service | GitHub Repo (Fork) | Docker Hub Image |
| --- | --- | --- |
| store-front | [store-front-L8](https://github.com/HananAli89/store-front-L8) | [hananali89/store-front:lab8](https://hub.docker.com/r/hananali89/store-front/tags) |
| store-admin | [store-admin-L8](https://github.com/HananAli89/store-admin-L8) | [hananali89/store-admin:lab8](https://hub.docker.com/r/hananali89/store-admin/tags) |
| order-service | [order-service-L8](https://github.com/HananAli89/order-service-L8) | [hananali89/order-service:lab8](https://hub.docker.com/r/hananali89/order-service/tags) |
| product-service | [product-service-L8](https://github.com/HananAli89/product-service-L8) | [hananali89/product-service:lab8](https://hub.docker.com/r/hananali89/product-service/tags) |
| makeline-service | [makeline-service-L8](https://github.com/HananAli89/makeline-service-L8) | [hananali89/makeline-service:lab8](https://hub.docker.com/r/hananali89/makeline-service/tags) |
| ai-service | [ai-service-L8](https://github.com/HananAli89/ai-service-L8) | [hananali89/ai-service:lab8](https://hub.docker.com/r/hananali89/ai-service/tags) |
| virtual-customer | [virtual-customer-L8](https://github.com/HananAli89/virtual-customer-L8) | [hananali89/virtual-customer:lab8](https://hub.docker.com/r/hananali89/virtual-customer/tags) |
| virtual-worker | [virtual-worker-L8](https://github.com/HananAli89/virtual-worker-L8) | [hananali89/virtual-worker:lab8](https://hub.docker.com/r/hananali89/virtual-worker/tags) |

**Deployment files:**
- [`aps-all-in-one-Task1.yaml`](aps-all-in-one-Task1.yaml): the six application services, MongoDB, and RabbitMQ
- [`admin-tasks.yaml`](admin-tasks.yaml): the Virtual Customer and Virtual Worker simulators

The AI service uses the direct OpenAI API (USE_AZURE_OPENAI: "False") with the class credentials provided through Brightspace, following the lab's Student Guide for OpenAI. The API keys are supplied through Kubernetes Secrets (secrets.yaml), which are not included in this repository.

## Task 2: Azure Managed Services to Replace MongoDB and RabbitMQ

In this deployment, MongoDB and RabbitMQ are self-hosted inside the AKS cluster as single-replica StatefulSets. Neither StatefulSet in `aps-all-in-one.yaml` defines `volumeClaimTemplates`, so their data is stored only in the Pod. If `mongodb-0` or `rabbitmq-0` is replaced, stored orders and queued messages are lost, and we are also responsible for patching, scaling, and backing up both services ourselves. Azure offers managed services that could replace both. These were researched for this task and not deployed in this lab.

### 1. Azure DocumentDB (replaces MongoDB)

- **Purpose:** A fully managed, MongoDB-compatible document database, formerly known as vCore-based Azure Cosmos DB for MongoDB. It is built on the open-source DocumentDB engine governed by the Linux Foundation [1].
- **Why it is a good fit:** It is compatible with existing MongoDB drivers and tools [2], so the `makeline-service` would keep using its MongoDB driver. Only the `ORDER_DB_URI` connection string would change from `mongodb://mongodb:27017` to the managed endpoint.
- **What problem it solves:** The order data in `orderdb` would no longer depend on a Pod's lifecycle. Azure handles the infrastructure, patching, and scaling, and the service includes free backups for up to 35 days [2]. This removes the data-loss risk of the current StatefulSet without volumes.
- **Alternative:** Azure Cosmos DB for MongoDB (RU) also implements the MongoDB wire protocol, but bills by throughput in Request Units, which suits variable or bursty workloads [3].

### 2. Azure Service Bus (replaces RabbitMQ)

- **Purpose:** A fully managed enterprise message broker with queues and publish-subscribe topics, used to decouple applications and services [4].
- **Why it is a good fit:** The pet store uses a simple queue pattern: `order-service` publishes orders to the `orders` queue, and `makeline-service` consumes them. A Service Bus queue provides the same pattern. Service Bus uses AMQP 1.0 as its primary protocol [5], and the cluster's RabbitMQ already enables the `rabbitmq_amqp1_0` plugin in `config-maps.yaml`.
- **What problem it solves:** Queued orders are stored by Azure outside the cluster, so restarting or rescheduling Pods cannot erase them. Service Bus can optionally span three availability zones [4], which removes the single point of failure of one RabbitMQ Pod. It also includes features such as dead-letter queues for messages that cannot be processed [4].

### Summary

| Self-hosted service | Azure managed replacement | Main benefit |
| --- | --- | --- |
| MongoDB (StatefulSet) | Azure DocumentDB | Durable, backed-up order data with minimal code changes |
| RabbitMQ (StatefulSet) | Azure Service Bus | Durable, highly available queue managed by Azure |

## References
1. Microsoft DevBlogs – Azure DocumentDB is now generally available: https://devblogs.microsoft.com/cosmosdb/azure-documentdb-is-now-generally-available/
2. Microsoft Azure – Azure DocumentDB: https://azure.microsoft.com/products/documentdb
3. Microsoft Learn – Azure Cosmos DB for MongoDB (RU): https://learn.microsoft.com/azure/cosmos-db/mongodb/ru/introduction
4. Microsoft Learn – What is Azure Service Bus?: https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview
5. Microsoft Learn – AMQP 1.0 in Azure Service Bus: https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-amqp-overview