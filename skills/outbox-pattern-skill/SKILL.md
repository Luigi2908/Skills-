---
name: outbox-pattern
description: Implements the Transactional Outbox pattern for reliable message publishing when updating a database. Ensures atomicity between DB writes and event/message delivery to brokers (RabbitMQ, Kafka). Use when implementing event-driven architectures, saga patterns, domain events, or any scenario requiring guaranteed delivery without 2PC. Prioritizes NestJS with TypeORM; includes patterns for polling publisher and transaction log tailing.
category: architecture
---

# Transactional Outbox Pattern

Implements the Transactional Outbox pattern to atomically update a database and publish messages/events—without distributed transactions (2PC).

## Use this skill when

- Publishing domain events after updating aggregates
- Participating in sagas or choreography patterns
- Sending messages to RabbitMQ, Kafka, or other brokers after DB commits
- Requiring guaranteed delivery without 2PC
- Building event-driven microservices

## Core principle

1. **Write to outbox in same transaction** as business data
2. **Message relay** (separate process) reads outbox and publishes to broker
3. **Order preserved**: messages sent in same order as DB commits
4. **At-least-once**: consumers must be idempotent (track processed IDs)

---

## NestJS Implementation (Priority)

### 1. Outbox entity (TypeORM)

```typescript
// src/outbox/outbox.entity.ts
import { Entity, PrimaryGeneratedColumn, Column, CreateDateColumn } from 'typeorm';

@Entity('outbox')
export class OutboxEntity {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  aggregateType: string;

  @Column()
  aggregateId: string;

  @Column()
  eventType: string;

  @Column('jsonb')
  payload: Record<string, unknown>;

  @Column({ default: 'pending' })
  status: 'pending' | 'published' | 'failed';

  @CreateDateColumn()
  createdAt: Date;
}
```

### 2. Write business data + outbox in one transaction

```typescript
// order.service.ts
@Injectable()
export class OrderService {
  constructor(
    @InjectRepository(Order) private orderRepo: Repository<Order>,
    @InjectRepository(OutboxEntity) private outboxRepo: Repository<OutboxEntity>,
    private dataSource: DataSource,
  ) {}

  async createOrder(dto: CreateOrderDto): Promise<Order> {
    return this.dataSource.transaction(async (manager) => {
      const order = manager.create(Order, dto);
      await manager.save(Order, order);

      const outbox = manager.create(OutboxEntity, {
        aggregateType: 'Order',
        aggregateId: order.id,
        eventType: 'OrderCreated',
        payload: { orderId: order.id, ... },
        status: 'pending',
      });
      await manager.save(OutboxEntity, outbox);

      return order;
    });
  }
}
```

### 3. Message relay (polling publisher)

```typescript
// outbox-relay.service.ts
@Injectable()
export class OutboxRelayService implements OnModuleInit {
  constructor(
    @InjectRepository(OutboxEntity) private outboxRepo: Repository<OutboxEntity>,
    @Inject(AMQP_CONNECTION) private amqp: AmqpConnection,
  ) {}

  async onModuleInit() {
    this.startPolling();
  }

  private startPolling() {
    setInterval(() => this.publishPending(), 1000);
  }

  async publishPending() {
    const pending = await this.outboxRepo.find({
      where: { status: 'pending' },
      order: { createdAt: 'ASC' },
      take: 100,
    });

    for (const msg of pending) {
      try {
        await this.amqp.publish('orders', msg.eventType, msg.payload);
        await this.outboxRepo.update(msg.id, { status: 'published' });
      } catch (err) {
        await this.outboxRepo.update(msg.id, { status: 'failed' });
        // Add retry logic or dead-letter handling
      }
    }
  }
}
```

### 4. Module wiring

```typescript
// outbox.module.ts
@Module({
  imports: [
    TypeOrmModule.forFeature([OutboxEntity]),
  ],
  providers: [OutboxRelayService],
  exports: [TypeOrmModule],
})
export class OutboxModule {}
```

---

## Message relay options

| Approach | Use when | Pros | Cons |
|----------|----------|------|-----|
| **Polling publisher** | Simple setups | Easy to implement | Delay, DB load |
| **Transaction log tailing** | High throughput | No polling, low latency | Requires Debezium/CDC |

For polling: use indexed `status` + `createdAt`, batch size 50–100, interval 500ms–2s. For transaction log tailing, see [reference.md](reference.md).

---

## Checklist

- [ ] Outbox table in same DB as business data
- [ ] Insert outbox row in same transaction as business write
- [ ] Relay marks published only after broker ack
- [ ] Consumers are idempotent (e.g., by message ID)
- [ ] On startup, relay processes any leftover pending messages

---

## Additional resources

- For detailed NestJS examples (RabbitMQ, Kafka, retries), see [examples.md](examples.md)
- For transaction log tailing and polling comparison, see [reference.md](reference.md)
