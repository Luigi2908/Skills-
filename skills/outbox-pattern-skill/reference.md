# Outbox Pattern Reference

## Message relay strategies

### Polling publisher

- **How it works**: Scheduled job queries the outbox table for `status = 'pending'`, publishes to broker, then marks as `published`.
- **Index**: `CREATE INDEX idx_outbox_pending ON outbox (status, created_at) WHERE status = 'pending'`
- **Settings**: Batch 50–100, interval 500ms–2s, avoid overlapping runs.

### Transaction log tailing (CDC)

- **How it works**: Tail the DB transaction log (e.g. WAL) and emit changes to Kafka. Tools: Debezium, pg_logical.
- **Pros**: No polling, lower latency, scales well.
- **Cons**: Requires CDC tooling and separate infra.

---

## Related patterns

- **Polling publisher**: [microservices.io](https://microservices.io/patterns/data/polling-publisher.html)
- **Transaction log tailing**: [microservices.io](https://microservices.io/patterns/data/transaction-log-tailing.html)
- **Saga pattern**: Often uses outbox for saga step events

---

## Consumer requirements

1. **Idempotency** – Handle duplicate deliveries (track `eventId` or `correlationId`).
2. **Order** – If needed, process by `aggregateId` or partition key.
