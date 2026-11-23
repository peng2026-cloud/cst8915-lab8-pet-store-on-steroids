# CST8915 Lab 8: Deploying and Managing the Algonquin Pet Store (On Steroids)

| Student Name | Student ID | Course | Semester |
|---|---|---|---|
| Peng Wang | 041107730 | CST8915 Full-stack Cloud-native Development | Fall 2026 |

🎥 [Watch Demo Video](https://www.youtube.com/watch?v=0LB91fKwM8U)

---

## 1. Demo Video

The video deploys the application to AKS with the updated [`aps-all-in-one-Task1.yaml`](aps-all-in-one-Task1.yaml). All commands run in this repository's folder.

### 1.1 Show the Cluster Nodes

One System node (`masterpool`) and two worker nodes (`workerspool`) are `Ready`.

```bash
kubectl get nodes
```

### 1.2 Deploy the ConfigMap and Secrets

`secrets.yaml` holds the OpenAI keys from Brightspace. It is listed in `.gitignore` and is not part of this repository.

```bash
kubectl apply -f config-maps.yaml
```

```bash
kubectl apply -f secrets.yaml
```

```bash
kubectl get configmap rabbitmq-enabled-plugins
```

```bash
kubectl get secret openai-gpt-api-secret openai-dalle-api-secret
```

### 1.3 Deploy the Application

```bash
kubectl apply -f aps-all-in-one-Task1.yaml
```

### 1.4 Show the Pods

Run the command again until every Pod is `Running`.

```bash
kubectl get pods
```

### 1.5 Verify the Image Names

The `IMAGES` column shows the six application services running the `zhizunbao2019/*:lab8` images.

```bash
kubectl get deployments,statefulsets -o wide
```

### 1.6 Show the Services

`store-front` and `store-admin` each have an `EXTERNAL-IP`.

```bash
kubectl get services
```

### 1.7 Place an Order in the Store Front

Open `http://<store-front EXTERNAL-IP>`, add a product to the cart and check out.

### 1.8 Add a Product with the AI Assistant

1. Open `http://<store-admin EXTERNAL-IP>`, go to **Products** and add a product.
2. Enter the name `Feather Wand Cat Toy`, the price `12.99` and the keywords `cat, toy, feather, interactive`.
3. Select **Ask AI Assistant**, then **Generate Image**, and save the product.
4. Refresh the Store Front and open the new product. The generated description and photo appear.

### 1.9 Deploy Virtual Customer and Worker

```bash
kubectl apply -f admin-tasks.yaml
```

Both simulators run the `zhizunbao2019/*:lab8` images.

```bash
kubectl get deployments virtual-customer virtual-worker -o wide
```

### 1.10 Show the Simulator Logs

```bash
kubectl logs deployment/virtual-customer --tail=5
```

```bash
kubectl logs deployment/virtual-worker --tail=5
```

### 1.11 Show the Orders in the Store Admin

In the Store Admin, open **Orders**. The orders placed by the virtual customer appear and are completed by the virtual worker.

### 1.12 Scale the Order Service

```bash
kubectl scale deployment order-service --replicas=3
```

```bash
kubectl get pods -l app=order-service
```

```bash
kubectl top nodes
```

## 2. Azure Managed Services

| # | Self-hosted service | Azure managed service |
|---|---|---|
| 1 | MongoDB (`mongodb` StatefulSet) | Azure DocumentDB (with MongoDB compatibility) |
| 2 | RabbitMQ (`rabbitmq` StatefulSet) | Azure Service Bus |

### 2.1 Azure DocumentDB (with MongoDB compatibility) for MongoDB

**Name and purpose.** Azure DocumentDB is “a fully managed, MongoDB-compatible database on Azure” ([Microsoft, n.d.-a, “What is Azure DocumentDB (with MongoDB compatibility)?” section, para. 1](https://learn.microsoft.com/en-us/azure/documentdb/overview#:~:text=a%20fully%20managed%2C%20MongoDB%2Dcompatible%20database%20on%20Azure)). It was “previously named vCore-based Azure Cosmos DB for MongoDB” ([Microsoft, n.d.-a, “Naming and history” section, list item 1](https://learn.microsoft.com/en-us/azure/documentdb/overview#:~:text=was%20previously%20named%20vCore%2Dbased%20Azure%20Cosmos%20DB%20for%20MongoDB)). In this application it would store the `orders` collection of `orderdb`, which `makeline-service` reads and writes.

**Why it is a good fit.** Microsoft states that “Your existing MongoDB drivers, tools, and application code connect without changes” ([Microsoft, n.d.-a, “What is Azure DocumentDB (with MongoDB compatibility)?” section, para. 1](https://learn.microsoft.com/en-us/azure/documentdb/overview#:~:text=Your%20existing%20MongoDB%20drivers%2C%20tools%2C%20and%20application%20code%20connect%20without%20changes)). `makeline-service` already uses the MongoDB driver and reads its connection from `ORDER_DB_URI`, so the change is the connection string, user name and password in `aps-all-in-one.yaml`, and the `mongodb` StatefulSet is removed.

**Problem it solves.** The `mongodb` StatefulSet keeps its data on one Pod and one disk in the cluster, and the team has to patch, upgrade and back it up. For self-hosted MongoDB, Microsoft lists “No patching, upgrades, replica set management, or backup scripts. A 99.995% availability SLA.” ([Microsoft, n.d.-a, “Who uses Azure DocumentDB (with MongoDB compatibility)?” section, table row 3](https://learn.microsoft.com/en-us/azure/documentdb/overview#:~:text=No%20patching%2C%20upgrades%2C%20replica%20set%20management%2C%20or%20backup%20scripts)).

### 2.2 Azure Service Bus for RabbitMQ

**Name and purpose.** Azure Service Bus is “a fully managed enterprise message broker with message queues and publish-subscribe topics” ([Microsoft, n.d.-b, “What is Azure Service Bus?” section, para. 1](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview#:~:text=fully%20managed%20enterprise%20message%20broker)). In this application it would hold the `orders` queue between `order-service`, which sends orders, and `makeline-service`, which receives them.

**Why it is a good fit.** Both services already talk to RabbitMQ through its AMQP 1.0 plugin (`rabbitmq_amqp1_0` in `config-maps.yaml`), and “The primary wire protocol for Service Bus is Advanced Messaging Queueing Protocol (AMQP) 1.0” ([Microsoft, n.d.-b, “Compliance with standards and protocols” section, para. 1](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview#:~:text=The%20primary%20wire%20protocol%20for%20Service%20Bus%20is)). `order-service` uses the `rhea` library, which Microsoft lists as the Service Bus client for JavaScript/Node ([Microsoft, n.d.-b, “Client libraries” section, table row 7](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview#:~:text=Rhea)). The READMEs of [`order-service`](https://github.com/peng2026-cloud/order-service-L8#option-2-azure-service-bus) and [`makeline-service`](https://github.com/peng2026-cloud/makeline-service-L8#option-2-azure-service-bus) already describe “Option 2: Azure Service Bus”, so the change is the queue host name, user name and password in `aps-all-in-one.yaml`, and the `rabbitmq` StatefulSet and its ConfigMap are removed.

**Problem it solves.** The `rabbitmq` StatefulSet is a single broker in the cluster: while its Pod restarts, `order-service` has nowhere to send orders, and the team has to run, upgrade and secure it. With Service Bus, “Producer and consumer don't have to be online or readily available at the same time” ([Microsoft, n.d.-b, “Overview” section, list item 2](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview#:~:text=Producer%20and%20consumer%20don%27t%20have%20to%20be%20online)), and once the broker accepts a message, “it always holds the message durably in triple-redundant storage” ([Microsoft, n.d.-b, “Queues” section, para. 2](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview#:~:text=holds%20the%20message%20durably%20in%20triple%2Dredundant%20storage)).

## 3. Updated `aps-all-in-one-Task1.yaml`

File: [`aps-all-in-one-Task1.yaml`](aps-all-in-one-Task1.yaml). Each image was built from the forked repository and pushed to Docker Hub.

| # | Service | Forked repository | Image |
|---|---|---|---|
| 1 | `order-service` | [peng2026-cloud/order-service-L8](https://github.com/peng2026-cloud/order-service-L8) | [`zhizunbao2019/order-service:lab8`](https://hub.docker.com/r/zhizunbao2019/order-service/tags) |
| 2 | `makeline-service` | [peng2026-cloud/makeline-service-L8](https://github.com/peng2026-cloud/makeline-service-L8) | [`zhizunbao2019/makeline-service:lab8`](https://hub.docker.com/r/zhizunbao2019/makeline-service/tags) |
| 3 | `product-service` | [peng2026-cloud/product-service-L8](https://github.com/peng2026-cloud/product-service-L8) | [`zhizunbao2019/product-service:lab8`](https://hub.docker.com/r/zhizunbao2019/product-service/tags) |
| 4 | `store-front` | [peng2026-cloud/store-front-L8](https://github.com/peng2026-cloud/store-front-L8) | [`zhizunbao2019/store-front:lab8`](https://hub.docker.com/r/zhizunbao2019/store-front/tags) |
| 5 | `store-admin` | [peng2026-cloud/store-admin-L8](https://github.com/peng2026-cloud/store-admin-L8) | [`zhizunbao2019/store-admin:lab8`](https://hub.docker.com/r/zhizunbao2019/store-admin/tags) |
| 6 | `ai-service` | [peng2026-cloud/ai-service-L8](https://github.com/peng2026-cloud/ai-service-L8) | [`zhizunbao2019/ai-service:lab8`](https://hub.docker.com/r/zhizunbao2019/ai-service/tags) |

## 4. Updated `admin-tasks.yaml`

File: [`admin-tasks.yaml`](admin-tasks.yaml).

| # | Service | Forked repository | Image |
|---|---|---|---|
| 1 | `virtual-customer` | [peng2026-cloud/virtual-customer-L8](https://github.com/peng2026-cloud/virtual-customer-L8) | [`zhizunbao2019/virtual-customer:lab8`](https://hub.docker.com/r/zhizunbao2019/virtual-customer/tags) |
| 2 | `virtual-worker` | [peng2026-cloud/virtual-worker-L8](https://github.com/peng2026-cloud/virtual-worker-L8) | [`zhizunbao2019/virtual-worker:lab8`](https://hub.docker.com/r/zhizunbao2019/virtual-worker/tags) |

## 5. Notes

| # | Item | Detail |
|---|---|---|
| 1 | Cluster | `AlgonquinPetStoreCluster` in West US: `masterpool` (System, 1 × Standard_E2s_v3) and `workerspool` (User, 2 × Standard_D2s_v3). The Lab 6 size Standard_D2as_v4 is available to this subscription only in East US, where its quota of 4 vCPUs fits only two nodes. |
| 2 | AI backing service | The direct OpenAI path from `Student_Guide_OpenAI.md`, with the Brightspace `secrets.yaml` (not committed). |

## References

Microsoft. (n.d.-a). *What is Azure DocumentDB (with MongoDB compatibility)?* Microsoft Learn. https://learn.microsoft.com/en-us/azure/documentdb/overview

Microsoft. (n.d.-b). *What is Azure Service Bus?* Microsoft Learn. https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview

---

GenAI declaration: As permitted for labs in this course, Claude (Anthropic) was used to explain the steps, check portal settings and terminal output, build and push the images, and draft this README. The portal steps, commands and demo recording were done by the student.
