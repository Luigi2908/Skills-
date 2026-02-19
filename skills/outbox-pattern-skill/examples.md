# Outbox Pattern – Examples

NestJS-focused examples for the Transactional Outbox pattern.

---

## Example 1: Order Service with RabbitMQ

```typescript
// order.module.ts
@Module({
  imports: [
    TypeOrmModule.forFeature([Order, OutboxEntity]),
    OutboxModule,
  ],
  controllers: [OrderController],
  providers: [OrderService],
})
export class OrderModule {}
```

```typescript
// order.service.ts
@Injectable()
export class OrderService {
  constructor(
    @InjectRepository(Order) private orderRepo: Repository<Order>,
    @InjectRepository(OutboxEntity) private outboxRepo: Repository<OutboxEntity>,
    private dataSource: DataSource,
  ) {}

  async placeOrder(dto: PlaceOrderDto): Promise<Order> {
    return this.dataSource.transaction(async (em) => {
      const order = em.create(Order, {
        ...dto,
        status: 'PENDING',
      });
      await em.save(Order, order);

      await em.save(OutboxEntity, em.create(OutboxEntity, {
        aggregateType: 'Order',
        aggregateId: order.id,
        eventType: 'order.placed',
        payload: {
          orderId: order.id,
          customerId: dto.customerId,
          total: dto.total,
        },
      }));

      return order;
    });
  }
}
```

---

## Example 2: Relay with RabbitMQ (BullMQ-style)

```typescript
// outbox-relay.service.ts
import { Injectable, OnModuleInit } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { InjectAmqpConnection } from '@golevelup/nestjs-rabbitmq';
import { AmqpConnection } from '@golevelup/nestjs-rabbitmq';
import { OutboxEntity } from './outbox.entity';

@Injectable()
export class OutboxRelayService implements OnModuleInit {
  private readonly BATCH_SIZE = 50;
  private readonly INTERVAL_MS = 1000;

  constructor(
    @InjectRepository(OutboxEntity) private outbox: Repository<OutboxEntity>,
    private amqp: AmqpConnection,
  ) {}

  onModuleInit() {
    setInterval(() => this.processPending(), this.INTERVAL_MS);
  }

  private async processPending() {
    const items = await this.outbox.find({
      where: { status: 'pending' },
      order: { createdAt: 'ASC' },
      take: this.BATCH_SIZE,
    });

    for (const item of items) {
      try {
        await this.amqp.publish('events', item.eventType, item.payload, {
          correlationId: item.id,
        });
        await this.outbox.update(item.id, { status: 'published' });
      } catch (e) {
        await this.outbox.update(item.id, { status: 'failed' });
      }
    }
  }
}
```

---

## Example 3: Consumer Idempotency

```typescript
// order-events.handler.ts
@Injectable()
export class OrderEventsHandler {
  constructor(
    @InjectRepository(ProcessedEvent) private processed: Repository<ProcessedEvent>,
    private inventoryService: InventoryService,
  ) {}

  @RabbitSubscribe({
    exchange: 'events',
    routingKey: 'order.placed',
    queue: 'inventory-order-placed',
  })
  async handleOrderPlaced(msg: OrderPlacedEvent, amqpMsg: Message) {
    const eventId = amqpMsg.properties?.correlationId as string;

    const existing = await this.processed.findOne({ where: { eventId } });
    if (existing) return; // Already processed

    await this.inventoryService.reserve(msg.orderId, msg.items);
    await this.processed.save({ eventId });
  }
}
```

---

## Example 4: Kafka with NestJS

```typescript
// kafka-outbox-relay.service.ts
@Injectable()
export class KafkaOutboxRelayService implements OnModuleInit {
  constructor(
    @InjectRepository(OutboxEntity) private outbox: Repository<OutboxEntity>,
    @Inject('KAFKA_PRODUCER') private producer: { send: (args: object) => Promise<unknown> },
  ) {}

  async processPending() {
    const items = await this.outbox.find({
      where: { status: 'pending' },
      order: { createdAt: 'ASC' },
      take: 100,
    });

    for (const item of items) {
      try {
        await this.producer.send({
          topic: 'domain-events',
          messages: [{
            key: item.aggregateId,
            value: JSON.stringify({ ...item.payload, _eventId: item.id }),
            headers: { eventType: item.eventType },
          }],
        });
        await this.outbox.update(item.id, { status: 'published' });
      } catch (e) {
        await this.outbox.update(item.id, { status: 'failed' });
      }
    }
  }
}
```

---

## Example 5: Helper for reusable outbox insert

```typescript
// outbox.helper.ts
export function createOutboxEntry<T>(
  aggregateType: string,
  aggregateId: string,
  eventType: string,
  payload: T,
): Partial<OutboxEntity> {
  return {
    aggregateType,
    aggregateId,
    eventType,
    payload: payload as Record<string, unknown>,
    status: 'pending',
  };
}

// Usage in service
await em.save(OutboxEntity, em.create(OutboxEntity,
  createOutboxEntry('Order', order.id, 'order.placed', { orderId: order.id, ... })
));
```
