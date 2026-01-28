# HMS Backend - Event-Bus Redis Architecture

## Overview

The HMS Backend has been migrated to a dedicated event-bus Redis architecture to enable proper inter-service communication and prepare for future RabbitMQ migration. This replaces the previous per-service Redis setup with a single shared event-bus Redis for all event publishing and consumption.

## Architecture Changes

### Before (Per-Service Redis)
- Each microservice had its own Redis container for both caching and event queuing
- Event consumption was limited to per-service boundaries
- Complex cross-service event routing required multiple Redis instances
- Resource duplication and management overhead

### After (Shared Event-Bus Redis)
- Single shared event-bus Redis (port 6384) for all inter-service events
- Separate cache Redis (port 6379) for caching needs where required
- Centralized event consumption by notification-service
- Clean separation of concerns between event bus and caching
- Future-ready for RabbitMQ migration

## Infrastructure Setup

### Root Level Docker Compose
```yaml
version: '3.8'

services:
  event-bus-redis:
    image: redis:7-alpine
    container_name: hms_event_bus_redis
    restart: always
    ports:
      - '6384:6379'
    volumes:
      - event_bus_redis_data:/data
    command: redis-server --appendonly yes

  cache-redis:
    image: redis:7-alpine
    container_name: cache-redis
    restart: always
    ports:
      - '6379:6379'
    volumes:
      - cache_data:/data
    command: redis-server --appendonly yes

volumes:
  event_bus_redis_data:
  cache_data:
```

### Service Configuration Updates
All services now use two Redis URLs:
- `EVENT_BUS_REDIS_URL`: For event publishing/consumption (redis://localhost:6384)
- `CACHE_REDIS_URL`: For caching needs (redis://localhost:6379)

### Event Queue Updates
- All `eventQueue.js` files updated to use `config.eventBusRedisUrl`
- Queue names standardized (e.g., 'user-events', 'booking-events', 'worker-events')
- Notification-service uses 'notification-events' for consumption

## Event Publishing Integration

### User Service Events
The user-service now publishes events for all CRUD operations:

- `user.registered`: When a new user registers
- `user.verified`: When email verification completes
- `user.updated`: When user profile is updated
- `user.deleted`: When user account is deleted

### Event Payload Structure
```json
{
  "userId": "uuid",
  "email": "user@example.com",
  "role": "customer|worker|admin",
  "timestamp": "2024-01-01T00:00:00.000Z"
}
```

## Environment Variables

Set the following environment variables for all services:

```bash
EVENT_BUS_REDIS_URL=redis://localhost:6384
CACHE_REDIS_URL=redis://localhost:6379
```

## Migration Benefits

1. **Simplified Event Routing**: Single event bus for all cross-service communication
2. **Resource Efficiency**: Reduced Redis container overhead
3. **Scalability**: Centralized event processing enables better monitoring and scaling
4. **Future-Proofing**: Clean abstraction layer for RabbitMQ migration
5. **Maintainability**: Consistent event handling patterns across services

## Testing the Event Flow

1. Start the shared Redis services:
   ```bash
   docker-compose -f docker-compose.yml up -d
   ```

2. Start notification-service to consume events:
   ```bash
   cd notification-service && npm start
   ```

3. Start user-service and perform operations:
   ```bash
   cd user-service && npm start
   # Register a user, verify email, update profile
   ```

4. Verify events are consumed in notification-service logs

## Rollback Plan

If issues arise, individual service Redis can be restored by:
1. Reverting config.js files to use single `redisUrl`
2. Restoring individual Redis containers in service docker-compose.yml files
3. Updating environment variables back to `REDIS_URL`

## Future RabbitMQ Migration

The shared event-bus architecture provides a clean migration path:
1. Replace Bull queues with AMQP publishers
2. Update eventQueue.js to use RabbitMQ connections
3. Maintain same event payload structures
4. Update notification-service consumer to use RabbitMQ

## Monitoring and Observability

- Monitor event-bus Redis for queue lengths and processing rates
- Log event publishing and consumption for debugging
- Set up alerts for failed event processing
- Track event throughput for capacity planning
