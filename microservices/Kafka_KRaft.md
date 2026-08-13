# Apache Kafka — Complete Staff-Level Guide
*From zero to production · KRaft + Legacy Zookeeper · Zero-downtime migration*
*Code: Node.js (KafkaJS) + NestJS integration*

---

# Part 1: What Is Kafka and Why Does It Exist?

## The Problem Kafka Solves

```
WITHOUT Kafka (synchronous, tightly coupled):

  User places order → Order Service calls:
    → Payment Service (500ms)
    → Inventory Service (200ms)
    → Email Service (300ms)
    → Analytics Service (100ms)
  Total: 1100ms. User waits for ALL of them.
  
  If Email Service is DOWN → entire order fails.
  If Analytics Service is SLOW → user waits.
  Adding a new service → modify Order Service code.

WITH Kafka (asynchronous, decoupled):

  User places order → Order Service publishes "order.created" event → done (5ms)
  
  Kafka distributes the event to ALL interested services independently:
    → Payment Service picks it up (whenever ready)
    → Inventory Service picks it up
    → Email Service picks it up
    → Analytics Service picks it up
  
  If Email is down → message waits in Kafka until it's back.
  Adding a new service → just subscribe to the event. Zero code changes.
```

## Kafka Is NOT Just a Message Queue

```
RabbitMQ / BullMQ (traditional message queue):
  • Message delivered to ONE consumer, then DELETED
  • No replay — once consumed, gone forever
  • Simple routing (direct, fanout, topic)

Kafka (distributed event streaming platform):
  • Messages are PERSISTED on disk (days, weeks, forever)
  • Multiple consumers can read the SAME message independently
  • Consumers can REPLAY from any point in history
  • Ordered within a partition (guaranteed)
  • Millions of messages per second
  • Built for horizontal scaling across many servers

Think of it as:
  RabbitMQ = a post office (deliver letter, it's gone)
  Kafka = a newspaper archive (everyone reads it, it's kept forever)
```

## Core Concepts

```
TOPIC:
  A named stream of messages. Like a database table or a channel.
  Examples: "orders", "user-events", "payment-completed", "logs"
  
  Messages within a topic are IMMUTABLE (append-only, never modified).

PARTITION:
  A topic is split into partitions for parallelism.
  Each partition is an ordered, immutable sequence of messages.
  
  Topic "orders" with 3 partitions:
    Partition 0: [msg0, msg3, msg6, msg9, ...]
    Partition 1: [msg1, msg4, msg7, msg10, ...]
    Partition 2: [msg2, msg5, msg8, msg11, ...]
  
  ORDER is guaranteed WITHIN a partition, NOT across partitions.
  Messages with the same KEY always go to the SAME partition (consistent hashing).

OFFSET:
  Each message in a partition has a sequential ID called an offset.
  Partition 0: offset 0, 1, 2, 3, 4, ...
  
  Consumers track their position by offset:
    "I've read up to offset 47 in partition 0"
  
  This is how replay works — just reset offset to an earlier position.

BROKER:
  A Kafka server. A cluster has multiple brokers for fault tolerance.
  Each partition is stored on one broker (leader) with copies on others (replicas).

PRODUCER:
  Writes messages to topics.
  Chooses which partition via: key hashing, round-robin, or custom partitioner.

CONSUMER:
  Reads messages from topics.
  Tracks its position (offset) per partition.

CONSUMER GROUP:
  Multiple consumers working together. Each partition is assigned to exactly
  ONE consumer in the group. This is how Kafka parallelizes consumption.
  
  Topic with 6 partitions + Consumer Group with 3 consumers:
    Consumer A: reads partitions 0, 1
    Consumer B: reads partitions 2, 3
    Consumer C: reads partitions 4, 5
  
  If Consumer B dies:
    Consumer A: reads partitions 0, 1, 2  (rebalanced!)
    Consumer C: reads partitions 3, 4, 5
  
  Rule: partitions ≥ consumers. Extra consumers sit idle.
  6 partitions + 8 consumers → 6 active, 2 idle.
```

```
Visual: The Complete Flow

  Producer                    Kafka Cluster                    Consumers
  ────────                    ─────────────                    ─────────
                           ┌─────────────────┐
  Order Service ──msg──▶   │  Topic: orders   │   ──▶ Payment Service (Group: payments)
                           │  ┌─Partition 0──┐│   ──▶ Inventory Service (Group: inventory)
                           │  │ [0][1][2][3]  ││   ──▶ Analytics Service (Group: analytics)
                           │  └──────────────┘│
                           │  ┌─Partition 1──┐│   Each group reads ALL messages
                           │  │ [0][1][2]    ││   independently at its own pace.
                           │  └──────────────┘│   
                           │  ┌─Partition 2──┐│   Messages are NOT deleted after reading.
                           │  │ [0][1][2][3] ││   They stay for the retention period.
                           │  └──────────────┘│
                           └─────────────────┘
```

---

# Part 2: Zookeeper vs KRaft

## Legacy Architecture (Zookeeper)

```
Kafka originally needed Zookeeper for:
  • Electing which broker is the controller (cluster coordinator)
  • Storing topic/partition metadata (which broker owns which partition)
  • Tracking broker liveness (heartbeats)
  • Managing consumer group offsets (old consumers only)

Architecture:
  ┌──────────┐   ┌──────────┐   ┌──────────┐
  │ Zookeeper│   │ Zookeeper│   │ Zookeeper│   (3-5 ZK nodes)
  │  Node 1  │   │  Node 2  │   │  Node 3  │
  └────┬─────┘   └────┬─────┘   └────┬─────┘
       │              │              │
       └──────────────┼──────────────┘
                      │
  ┌──────────┐   ┌────┴─────┐   ┌──────────┐
  │  Broker 1│   │ Broker 2 │   │ Broker 3 │   (Kafka brokers)
  │(controller)  │          │   │          │
  └──────────┘   └──────────┘   └──────────┘

Problems with Zookeeper:
  • Separate system to deploy, monitor, and maintain
  • Metadata splits between ZK and brokers → consistency issues
  • Scaling limits (~200K partitions per cluster)
  • Recovery after controller failure is slow (full metadata reload from ZK)
  • Two different systems = two different failure modes
```

## KRaft Mode (The New Way — Kafka 3.3+)

```
KRaft = Kafka Raft. Kafka manages its OWN metadata using the Raft consensus protocol.
No external Zookeeper needed.

Architecture:
  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
  │  Controller  │   │  Controller  │   │  Controller  │  (KRaft controllers)
  │  + Broker 1  │   │  + Broker 2  │   │  + Broker 3  │  (can be combined or separate)
  └──────────────┘   └──────────────┘   └──────────────┘

  Metadata is stored in an internal Kafka topic: __cluster_metadata
  Raft consensus elects a leader controller (like Raft leader election).
  All metadata changes are replicated as events in this topic.

Benefits over Zookeeper:
  ✓ Single system to manage (no ZK cluster)
  ✓ Faster controller failover (~seconds vs ~minutes)
  ✓ Scales to millions of partitions
  ✓ Simpler deployment (fewer moving parts)
  ✓ Better consistency (metadata in one place)
  ✓ Faster startup and shutdown
  
KRaft is production-ready since Kafka 3.3 (2022).
Zookeeper is deprecated and will be removed in Kafka 4.0.
```

---

# Part 3: Docker Compose Setup

## KRaft Mode (Recommended — No Zookeeper)

```yaml
# docker-compose.yml — Single broker KRaft (development)
version: "3.8"
services:
  kafka:
    image: confluentinc/cp-kafka:7.6.0
    hostname: kafka
    ports:
      - "9092:9092"       # external clients
      - "29092:29092"     # internal (other containers)
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: "broker,controller"           # combined mode
      KAFKA_CONTROLLER_QUORUM_VOTERS: "1@kafka:9093"     # controller voter list
      KAFKA_LISTENERS: "PLAINTEXT://0.0.0.0:29092,CONTROLLER://0.0.0.0:9093,EXTERNAL://0.0.0.0:9092"
      KAFKA_ADVERTISED_LISTENERS: "PLAINTEXT://kafka:29092,EXTERNAL://localhost:9092"
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: "PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT"
      KAFKA_CONTROLLER_LISTENER_NAMES: "CONTROLLER"
      KAFKA_INTER_BROKER_LISTENER_NAME: "PLAINTEXT"
      KAFKA_LOG_DIRS: "/var/lib/kafka/data"
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"            # explicit topic creation
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      CLUSTER_ID: "MkU3OEVBNTcwNTJENDM2Qk"               # must be fixed for KRaft
    volumes:
      - kafka_data:/var/lib/kafka/data

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports: ["8080:8080"]
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092
    depends_on: [kafka]

volumes:
  kafka_data:
```

## Production Multi-Broker KRaft

```yaml
# docker-compose.prod.yml — 3-broker KRaft cluster
version: "3.8"
services:
  kafka-1:
    image: confluentinc/cp-kafka:7.6.0
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: "broker,controller"
      KAFKA_CONTROLLER_QUORUM_VOTERS: "1@kafka-1:9093,2@kafka-2:9093,3@kafka-3:9093"
      KAFKA_LISTENERS: "PLAINTEXT://0.0.0.0:29092,CONTROLLER://0.0.0.0:9093,EXTERNAL://0.0.0.0:9092"
      KAFKA_ADVERTISED_LISTENERS: "PLAINTEXT://kafka-1:29092,EXTERNAL://localhost:9092"
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: "PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT"
      KAFKA_CONTROLLER_LISTENER_NAMES: "CONTROLLER"
      KAFKA_INTER_BROKER_LISTENER_NAME: "PLAINTEXT"
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      KAFKA_NUM_PARTITIONS: 6
      KAFKA_LOG_RETENTION_HOURS: 168         # 7 days
      KAFKA_LOG_SEGMENT_BYTES: 1073741824    # 1GB per segment
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "false"
      CLUSTER_ID: "MkU3OEVBNTcwNTJENDM2Qk"
    volumes:
      - kafka1_data:/var/lib/kafka/data
    ports: ["9092:9092"]

  kafka-2:
    image: confluentinc/cp-kafka:7.6.0
    environment:
      KAFKA_NODE_ID: 2
      KAFKA_PROCESS_ROLES: "broker,controller"
      KAFKA_CONTROLLER_QUORUM_VOTERS: "1@kafka-1:9093,2@kafka-2:9093,3@kafka-3:9093"
      KAFKA_LISTENERS: "PLAINTEXT://0.0.0.0:29092,CONTROLLER://0.0.0.0:9093,EXTERNAL://0.0.0.0:9093"
      KAFKA_ADVERTISED_LISTENERS: "PLAINTEXT://kafka-2:29092,EXTERNAL://localhost:9093"
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: "PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT"
      KAFKA_CONTROLLER_LISTENER_NAMES: "CONTROLLER"
      KAFKA_INTER_BROKER_LISTENER_NAME: "PLAINTEXT"
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      CLUSTER_ID: "MkU3OEVBNTcwNTJENDM2Qk"
    volumes:
      - kafka2_data:/var/lib/kafka/data
    ports: ["9093:9093"]

  kafka-3:
    image: confluentinc/cp-kafka:7.6.0
    environment:
      KAFKA_NODE_ID: 3
      KAFKA_PROCESS_ROLES: "broker,controller"
      KAFKA_CONTROLLER_QUORUM_VOTERS: "1@kafka-1:9093,2@kafka-2:9093,3@kafka-3:9093"
      KAFKA_LISTENERS: "PLAINTEXT://0.0.0.0:29092,CONTROLLER://0.0.0.0:9093,EXTERNAL://0.0.0.0:9094"
      KAFKA_ADVERTISED_LISTENERS: "PLAINTEXT://kafka-3:29092,EXTERNAL://localhost:9094"
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: "PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT"
      KAFKA_CONTROLLER_LISTENER_NAMES: "CONTROLLER"
      KAFKA_INTER_BROKER_LISTENER_NAME: "PLAINTEXT"
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      CLUSTER_ID: "MkU3OEVBNTcwNTJENDM2Qk"
    volumes:
      - kafka3_data:/var/lib/kafka/data
    ports: ["9094:9094"]

volumes:
  kafka1_data:
  kafka2_data:
  kafka3_data:
```

## Legacy Zookeeper Setup (For Reference)

```yaml
# docker-compose.zk.yml — Old way with Zookeeper
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports: ["2181:2181"]

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    depends_on: [zookeeper]
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: "zookeeper:2181"        # ← connects to ZK
      KAFKA_ADVERTISED_LISTENERS: "PLAINTEXT://kafka:29092,EXTERNAL://localhost:9092"
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: "PLAINTEXT:PLAINTEXT,EXTERNAL:PLAINTEXT"
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports: ["9092:9092"]
```

---

# Part 4: Zero-Downtime Migration (Zookeeper → KRaft)

```
Migration happens in 3 phases without stopping the cluster:

PHASE 1: Enable KRaft controllers alongside Zookeeper (dual-write)
  • Deploy KRaft controller nodes
  • Both ZK and KRaft controllers run simultaneously
  • Kafka brokers still use Zookeeper as primary
  • KRaft controllers sync metadata from Zookeeper

PHASE 2: Switch brokers from Zookeeper to KRaft
  • Rolling restart: one broker at a time
  • Change broker config: remove zookeeper.connect, add controller.quorum.voters
  • Each broker reconnects to KRaft controllers instead of Zookeeper
  • Zero downtime: partitions rebalance automatically during rolling restart

PHASE 3: Decommission Zookeeper
  • All brokers now use KRaft
  • Zookeeper ensemble can be shut down
  • Remove ZK configuration from all broker configs
```

```bash
# Step 1: Generate migration metadata
kafka-metadata.sh snapshot \
  --cluster-id MkU3OEVBNTcwNTJENDM2Qk \
  --source zk \
  --zookeeper localhost:2181 \
  --output /tmp/kraft-migration

# Step 2: Start KRaft controllers with migration flag
# In server.properties for controller nodes:
process.roles=controller
controller.quorum.voters=100@controller1:9093,101@controller2:9093,102@controller3:9093
zookeeper.metadata.migration.enable=true    # enable migration mode

# Step 3: Rolling restart brokers with new config
# In server.properties for broker nodes:
# Remove: zookeeper.connect=localhost:2181
# Add:
controller.quorum.voters=100@controller1:9093,101@controller2:9093,102@controller3:9093
zookeeper.metadata.migration.enable=true

# Restart brokers one at a time (zero downtime):
# 1. Stop broker 1, update config, start broker 1
# 2. Wait for ISR to recover (all replicas in sync)
# 3. Stop broker 2, update config, start broker 2
# 4. Wait for ISR
# 5. Stop broker 3, update config, start broker 3

# Step 4: Finalize migration (disable ZK migration mode)
kafka-metadata.sh finalize --cluster-id MkU3OEVBNTcwNTJENDM2Qk

# Step 5: Shut down Zookeeper ensemble
```

---

# Part 5: Node.js Producer & Consumer (KafkaJS)

```bash
npm install kafkajs
```

## Producer

```javascript
// src/kafka/producer.ts
import { Kafka, CompressionTypes, logLevel } from "kafkajs";

const kafka = new Kafka({
  clientId: "order-service",
  brokers: ["localhost:9092"],
  connectionTimeout: 10000,
  retry: { retries: 5, initialRetryTime: 300 },
  logLevel: logLevel.WARN,
});

const producer = kafka.producer({
  allowAutoTopicCreation: false,      // always create topics explicitly
  transactionTimeout: 30000,
  idempotent: true,                   // exactly-once semantics (no duplicate messages)
});

// ── Connect once at app startup ──
await producer.connect();

// ── Send a single message ──
await producer.send({
  topic: "orders",
  messages: [
    {
      key: "user-123",                // messages with same key → same partition → ordered
      value: JSON.stringify({
        orderId: "order-456",
        userId: "user-123",
        items: [{ productId: "prod-1", quantity: 2 }],
        total: 4999,
        createdAt: new Date().toISOString(),
      }),
      headers: {
        "event-type": "order.created",
        "correlation-id": "req-789",
        "source": "order-service",
      },
    },
  ],
  compression: CompressionTypes.GZIP,  // compress for large messages
  acks: -1,                            // wait for ALL replicas to confirm (safest)
  timeout: 30000,
});

// ── Send batch (multiple topics/messages) ──
await producer.sendBatch({
  topicMessages: [
    {
      topic: "orders",
      messages: [{ key: "user-1", value: JSON.stringify(order1) }],
    },
    {
      topic: "audit-log",
      messages: [{ key: "user-1", value: JSON.stringify(auditEntry) }],
    },
  ],
  compression: CompressionTypes.GZIP,
  acks: -1,
});

// ── Disconnect at app shutdown ──
await producer.disconnect();
```

### Producer Acks Levels

```
acks: 0   Fire and forget. Don't wait for any confirmation.
          Fastest. Risk: messages can be LOST if broker crashes.

acks: 1   Wait for the LEADER replica to confirm.
          Balanced. Risk: if leader crashes before replication, message lost.

acks: -1  Wait for ALL in-sync replicas (ISR) to confirm.
(all)     Safest. No data loss as long as min.insync.replicas are alive.
          Slowest but required for financial/critical data.
```

## Consumer

```javascript
// src/kafka/consumer.ts
import { Kafka, logLevel } from "kafkajs";

const kafka = new Kafka({
  clientId: "payment-service",
  brokers: ["localhost:9092"],
  logLevel: logLevel.WARN,
});

const consumer = kafka.consumer({
  groupId: "payment-service-group",     // consumer group ID
  sessionTimeout: 30000,                // if no heartbeat for 30s → considered dead
  heartbeatInterval: 3000,              // heartbeat every 3s
  maxWaitTimeInMs: 5000,
  retry: { retries: 10, initialRetryTime: 300 },
});

await consumer.connect();

// ── Subscribe to topics ──
await consumer.subscribe({ topic: "orders", fromBeginning: false });
// fromBeginning: true → read from offset 0 (replay all history)
// fromBeginning: false → read from latest (only new messages)

// Subscribe to multiple topics
await consumer.subscribe({ topics: ["orders", "returns"], fromBeginning: false });

// Subscribe with regex
await consumer.subscribe({ topic: /order\..*/, fromBeginning: false });

// ── Process messages ──
await consumer.run({
  autoCommit: true,                     // auto-commit offsets
  autoCommitInterval: 5000,             // commit every 5 seconds
  autoCommitThreshold: 100,             // or every 100 messages
  
  eachMessage: async ({ topic, partition, message, heartbeat }) => {
    const event = JSON.parse(message.value.toString());
    const eventType = message.headers?.["event-type"]?.toString();
    const key = message.key?.toString();

    console.log(`[${topic}][P${partition}][O${message.offset}] ${eventType}: ${key}`);

    try {
      switch (eventType) {
        case "order.created":
          await processPayment(event);
          break;
        case "order.cancelled":
          await refundPayment(event);
          break;
        default:
          console.warn(`Unknown event type: ${eventType}`);
      }
    } catch (error) {
      // Don't throw — it would crash the consumer.
      // Instead: log, send to DLQ, alert.
      await sendToDeadLetterQueue(topic, message, error);
    }

    // For long-running processing, send heartbeats to prevent rebalancing
    await heartbeat();
  },
});

// ── Manual commit (for at-least-once processing) ──
await consumer.run({
  autoCommit: false,      // disable auto-commit
  eachMessage: async ({ topic, partition, message }) => {
    await processMessage(message);    // process FIRST
    
    // Commit AFTER successful processing
    await consumer.commitOffsets([{
      topic,
      partition,
      offset: (BigInt(message.offset) + 1n).toString(),  // commit NEXT offset
    }]);
  },
});

// ── Batch processing (higher throughput) ──
await consumer.run({
  eachBatch: async ({ batch, resolveOffset, heartbeat, commitOffsetsIfNecessary }) => {
    for (const message of batch.messages) {
      await processMessage(message);
      resolveOffset(message.offset);    // mark as processed
      await heartbeat();                 // keep alive during long batches
    }
    await commitOffsetsIfNecessary();
  },
});

// ── Graceful shutdown ──
process.on("SIGTERM", async () => {
  await consumer.disconnect();    // commits final offsets, leaves consumer group
  process.exit(0);
});
```

---

# Part 6: NestJS Integration

```typescript
// src/kafka/kafka.module.ts
import { Module } from "@nestjs/common";
import { ClientsModule, Transport } from "@nestjs/microservices";
import { KafkaProducerService } from "./kafka-producer.service";

@Module({
  imports: [
    ClientsModule.register([{
      name: "KAFKA_SERVICE",
      transport: Transport.KAFKA,
      options: {
        client: {
          clientId: "order-service",
          brokers: [process.env.KAFKA_BROKERS || "localhost:9092"],
        },
        producer: {
          allowAutoTopicCreation: false,
          idempotent: true,
        },
        consumer: {
          groupId: "order-service-group",
        },
      },
    }]),
  ],
  providers: [KafkaProducerService],
  exports: [KafkaProducerService],
})
export class KafkaModule {}
```

```typescript
// src/kafka/kafka-producer.service.ts
import { Injectable, Inject, OnModuleInit } from "@nestjs/common";
import { ClientKafka } from "@nestjs/microservices";

@Injectable()
export class KafkaProducerService implements OnModuleInit {
  constructor(@Inject("KAFKA_SERVICE") private kafka: ClientKafka) {}

  async onModuleInit() {
    await this.kafka.connect();
  }

  async emit(topic: string, key: string, data: any, headers?: Record<string, string>) {
    return this.kafka.emit(topic, {
      key,
      value: JSON.stringify(data),
      headers: {
        "event-type": topic,
        "timestamp": new Date().toISOString(),
        ...headers,
      },
    });
  }
}

// Usage in any service:
@Injectable()
export class OrdersService {
  constructor(private kafkaProducer: KafkaProducerService) {}

  async createOrder(dto: CreateOrderDto) {
    const order = await this.orderRepo.create(dto);
    
    await this.kafkaProducer.emit("order.created", order.userId, {
      orderId: order.id,
      userId: order.userId,
      total: order.total,
      items: order.items,
    });
    
    return order;
  }
}
```

```typescript
// src/kafka/kafka-consumer.controller.ts — Handle incoming events
import { Controller } from "@nestjs/common";
import { EventPattern, Payload, Ctx, KafkaContext } from "@nestjs/microservices";

@Controller()
export class KafkaConsumerController {
  
  @EventPattern("order.created")    // subscribes to this topic
  async handleOrderCreated(@Payload() data: any, @Ctx() context: KafkaContext) {
    const { offset, partition, topic } = context.getMessage();
    console.log(`[${topic}][P${partition}][O${offset}] Processing order: ${data.orderId}`);
    
    await this.paymentService.processPayment(data);
    
    // Commit offset (manual acknowledgment)
    await context.getConsumer().commitOffsets([{
      topic, partition, offset: (BigInt(offset) + 1n).toString(),
    }]);
  }

  @EventPattern("order.cancelled")
  async handleOrderCancelled(@Payload() data: any) {
    await this.paymentService.refund(data.orderId);
  }
}
```

```typescript
// main.ts — Hybrid app (HTTP + Kafka)
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Attach Kafka microservice
  app.connectMicroservice({
    transport: Transport.KAFKA,
    options: {
      client: {
        brokers: [process.env.KAFKA_BROKERS || "localhost:9092"],
      },
      consumer: {
        groupId: "payment-service-group",
        sessionTimeout: 30000,
      },
    },
  });
  
  await app.startAllMicroservices();   // start Kafka consumers
  await app.listen(3000);              // start HTTP server
}
```

---

# Part 7: Topic Management and Admin

```javascript
// src/kafka/admin.ts
const admin = kafka.admin();
await admin.connect();

// ── Create topics ──
await admin.createTopics({
  topics: [
    {
      topic: "orders",
      numPartitions: 6,             // scale consumers (max 6 parallel consumers)
      replicationFactor: 3,          // survive 2 broker failures
      configEntries: [
        { name: "retention.ms", value: "604800000" },    // 7 days retention
        { name: "cleanup.policy", value: "delete" },      // or "compact" for latest-value
        { name: "min.insync.replicas", value: "2" },
        { name: "compression.type", value: "gzip" },
      ],
    },
    {
      topic: "orders.dlq",           // dead letter queue
      numPartitions: 3,
      replicationFactor: 3,
    },
    {
      topic: "user-profiles",
      numPartitions: 3,
      replicationFactor: 3,
      configEntries: [
        { name: "cleanup.policy", value: "compact" },     // keep latest value per key
        // Compacted topic: Kafka keeps only the LATEST message per key.
        // Key "user-123" → only the most recent profile is kept.
        // Like a key-value store that's also a change log.
      ],
    },
  ],
});

// ── List topics ──
const topics = await admin.listTopics();

// ── Describe topic ──
const metadata = await admin.fetchTopicMetadata({ topics: ["orders"] });

// ── Delete topic ──
await admin.deleteTopics({ topics: ["old-topic"] });

// ── List consumer groups ──
const groups = await admin.listGroups();

// ── Describe consumer group (see lag) ──
const description = await admin.describeGroups(["payment-service-group"]);

// ── Reset consumer group offsets ──
await admin.resetOffsets({
  groupId: "payment-service-group",
  topic: "orders",
  earliest: true,          // reset to beginning (replay all)
  // or: latest: true      // skip to end (discard backlog)
});

await admin.disconnect();
```

---

# Part 8: Dead Letter Queue (DLQ) Pattern

```javascript
// When a message can't be processed after retries → send to DLQ

async function processWithDLQ(message, topic) {
  const maxRetries = 3;
  const retryCount = parseInt(message.headers?.["retry-count"]?.toString() || "0");
  
  try {
    await processMessage(message);
  } catch (error) {
    if (retryCount < maxRetries) {
      // Retry: republish with incremented retry count
      await producer.send({
        topic,                          // same topic
        messages: [{
          key: message.key,
          value: message.value,
          headers: {
            ...message.headers,
            "retry-count": String(retryCount + 1),
            "last-error": error.message,
          },
        }],
      });
    } else {
      // Exhausted retries → send to DLQ
      await producer.send({
        topic: `${topic}.dlq`,
        messages: [{
          key: message.key,
          value: message.value,
          headers: {
            ...message.headers,
            "original-topic": topic,
            "failure-reason": error.message,
            "failed-at": new Date().toISOString(),
          },
        }],
      });
      console.error(`Message sent to DLQ: ${message.key}`, error);
      // Alert monitoring system (PagerDuty, Slack)
    }
  }
}
```

---

# Part 9: Exactly-Once Semantics and Transactions

```javascript
// Transactional producer: all messages in a transaction succeed or all fail

const producer = kafka.producer({
  idempotent: true,                 // required for transactions
  transactionalId: "order-processor-1",  // unique per producer instance
  maxInFlightRequests: 1,
});

await producer.connect();

// Atomic: read from orders → process → write to payments + audit
const transaction = await producer.transaction();

try {
  await transaction.send({
    topic: "payments",
    messages: [{ key: "order-1", value: JSON.stringify(payment) }],
  });

  await transaction.send({
    topic: "audit-log",
    messages: [{ key: "order-1", value: JSON.stringify(auditEntry) }],
  });

  // Commit consumer offset as part of the transaction
  await transaction.sendOffsets({
    consumerGroupId: "order-processor-group",
    topics: [{
      topic: "orders",
      partitions: [{ partition: 0, offset: "42" }],
    }],
  });

  await transaction.commit();   // all or nothing
} catch (error) {
  await transaction.abort();    // rollback everything
  throw error;
}
```

---

# Part 10: Production Configuration and Monitoring

## Key Broker Settings

```properties
# Performance
num.network.threads=8                  # threads for network requests
num.io.threads=16                      # threads for disk I/O
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400

# Durability
default.replication.factor=3           # copies of each partition
min.insync.replicas=2                  # must ack before producer gets success
unclean.leader.election.enable=false   # don't elect out-of-sync replica as leader

# Retention
log.retention.hours=168                # 7 days (or log.retention.ms)
log.segment.bytes=1073741824           # 1GB per segment file
log.cleanup.policy=delete             # or "compact" for key-value topics

# Quotas
replica.fetch.max.bytes=10485760       # max message size for replication
message.max.bytes=10485760             # max message size (10MB)
```

## Monitoring (Key Metrics)

```
BROKER METRICS:
  UnderReplicatedPartitions:  should be 0. >0 means replicas falling behind.
  ActiveControllerCount:      should be 1. 0 = no controller = cluster broken.
  OfflinePartitionsCount:     should be 0. >0 = data unavailable.
  RequestsPerSec:             throughput.
  RequestLatencyMs:           <10ms is healthy.
  
PRODUCER METRICS:
  record-send-rate:           messages/sec being produced.
  record-error-rate:          should be 0. >0 = delivery failures.
  request-latency-avg:        <50ms is healthy.

CONSUMER METRICS:
  records-lag-max:            THE most important consumer metric.
                              How many messages behind is the consumer?
                              Growing lag = consumer can't keep up.
  records-consumed-rate:      messages/sec being consumed.
  commit-latency-avg:         offset commit speed.

CONSUMER LAG:
  Lag = latest offset in partition - consumer's committed offset
  Lag = 0 → real-time. Lag growing → consumer needs more instances or faster processing.
```

```bash
# Check consumer lag from CLI
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group payment-service-group --describe

# Output:
# TOPIC     PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# orders    0          1000            1050            50
# orders    1          980             985             5
# orders    2          1100            1100            0
```

---

# Part 11: Event-Driven Architecture Patterns

```
PATTERN 1: Event Notification (fire and forget)
  Order Service → publishes "order.created" → other services react
  The event contains minimal data (just IDs).
  Consumers call back to get full details if needed.

PATTERN 2: Event-Carried State Transfer (full data in event)
  Order Service → publishes "order.created" with FULL order data
  Consumers have everything they need — no callbacks required.
  Pro: decoupled. Con: large messages, data duplication.

PATTERN 3: Event Sourcing (events ARE the database)
  Instead of storing current state, store the HISTORY of events.
  Current state = replay all events from the beginning.
  
  Events: [AccountCreated, Deposited $100, Withdrawn $30, Deposited $50]
  Balance: $0 + $100 - $30 + $50 = $120

PATTERN 4: CQRS (Command Query Responsibility Segregation)
  Writes go through Kafka → stored in write DB (event store).
  A consumer builds a READ-optimized view (denormalized, indexed).
  
  Write: POST /orders → Kafka → PostgreSQL (normalized)
  Read: GET /orders → MongoDB/Elasticsearch (denormalized, fast)

PATTERN 5: Saga (Distributed Transactions)
  Order created → Payment charged → Inventory reserved → Shipping scheduled
  
  If Inventory fails:
    → Publish "inventory.failed"
    → Payment service subscribes → refunds the charge (compensating transaction)
    → Order service subscribes → marks order as failed
  
  Each step publishes success/failure. Other services react accordingly.
  No distributed locks. No 2-phase commit. Just events and compensations.
```

---

# Part 12: 🧩 Interview Q&A

**Q: What is Kafka and how is it different from RabbitMQ?**
A: Kafka is a distributed event streaming platform. Messages are persisted to disk and retained for a configurable period (days/weeks). Multiple consumer groups can read the same messages independently. Consumers can replay from any offset. RabbitMQ is a traditional message broker — messages are delivered to one consumer and deleted. Kafka scales horizontally to millions of messages/sec via partitions. Use Kafka for event-driven architectures, event sourcing, log aggregation, stream processing. Use RabbitMQ for simple task queues with routing.

**Q: What is a consumer group and how do partitions relate to consumers?**
A: A consumer group is a set of consumers that cooperate to consume a topic. Each partition is assigned to exactly one consumer in the group. This enables parallel consumption — 6 partitions with 3 consumers means each consumer handles 2 partitions. If a consumer dies, its partitions are rebalanced to remaining consumers. More consumers than partitions means some sit idle. More partitions = higher parallelism but more overhead.

**Q: How does Kafka guarantee message ordering?**
A: Order is guaranteed only WITHIN a partition. Messages with the same key always go to the same partition (via consistent hashing), so all events for "user-123" are ordered. Across partitions there's no ordering guarantee. If you need global ordering, use a single partition (but lose parallelism). For most use cases, per-key ordering is sufficient.

**Q: Explain exactly-once semantics in Kafka.**
A: Three levels. At-most-once (acks=0): message may be lost, never duplicated. At-least-once (acks=-1, no transactions): message never lost, may be duplicated. Exactly-once: use idempotent producer (deduplicates retries) + transactions (atomic multi-partition writes) + consumer reads-committed isolation. The transactional producer groups sends and offset commits atomically — either all succeed or all roll back.

**Q: What is KRaft and why does it replace Zookeeper?**
A: KRaft uses the Raft consensus protocol for Kafka's internal metadata management, eliminating the external Zookeeper dependency. Benefits: single system to operate, faster controller failover (seconds vs minutes), scales to millions of partitions, simpler deployment. Kafka stores its own metadata in an internal __cluster_metadata topic. KRaft is production-ready since Kafka 3.3 and Zookeeper will be removed in Kafka 4.0.

**Q: How do you handle poison messages (messages that cause consumer crashes)?**
A: Dead Letter Queue (DLQ) pattern. Wrap message processing in try/catch. On failure, retry up to N times (with backoff). After exhausting retries, publish the message to a `<topic>.dlq` topic with error metadata (original topic, failure reason, timestamp). Monitor the DLQ size. Alert if it grows. Have a separate process or manual tool to inspect and replay DLQ messages after fixing the bug.

**Q: How would you migrate from Zookeeper to KRaft with zero downtime?**
A: Three phases. (1) Deploy KRaft controller nodes alongside existing Zookeeper, with `zookeeper.metadata.migration.enable=true`. Controllers sync metadata from ZK. (2) Rolling restart brokers one at a time, updating config to point to KRaft controllers instead of ZK. Wait for ISR to recover between each broker restart. (3) Run `kafka-metadata.sh finalize` to complete migration, then decommission Zookeeper. At no point is the cluster unavailable — partitions rebalance automatically during rolling restarts.
