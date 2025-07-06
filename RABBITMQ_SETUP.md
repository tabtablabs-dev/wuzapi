# RabbitMQ Setup for WuzAPI

WuzAPI can publish WhatsApp events to RabbitMQ for asynchronous processing and integration with other services.

## Configuration

To enable RabbitMQ integration, set the following environment variables:

```bash
# RabbitMQ connection URL (required)
RABBITMQ_URL=amqp://username:password@rabbitmq-host:5672/

# Queue name for events (optional, defaults to "whatsapp_events")
RABBITMQ_QUEUE=whatsapp_events
```

## Connection URL Format

The `RABBITMQ_URL` follows the AMQP URI specification:

```
amqp://[username[:password]@]host[:port][/vhost]
```

Examples:
- `amqp://guest:guest@localhost:5672/`
- `amqp://myuser:mypass@rabbitmq.example.com:5672/myvhost`
- `amqp://rabbitmq:5672/` (uses default guest credentials)

## Events Published

WuzAPI publishes the following WhatsApp events to RabbitMQ:

- **Connected**: Client successfully connected to WhatsApp
- **QR**: QR code generated for pairing
- **Message**: Message received (text, image, audio, document, video)
- **ReadReceipt**: Message marked as read
- **Presence**: User presence changed (online/offline)
- **HistorySync**: History sync completed
- **LoggedOut**: Client logged out
- **ChatPresence**: Chat presence received (e.g., typing)
- **Disconnected**: Client disconnected from WhatsApp
- **ConnectFailure**: Client failed to connect

## Message Format

Events are published as JSON messages with the following structure:

```json
{
  "event": "Message",
  "instanceId": "user-instance-id",
  "data": {
    // Event-specific data
  },
  "timestamp": "2024-01-01T12:00:00Z"
}
```

## Queue Configuration

By default, WuzAPI creates a durable queue with the following properties:
- **Durable**: Yes (survives broker restart)
- **Auto-delete**: No
- **Exclusive**: No

## Docker Compose Setup

### Using External RabbitMQ

If you have an existing RabbitMQ service, simply add the environment variables:

```yaml
services:
  wuzapi-server:
    environment:
      - RABBITMQ_URL=amqp://user:pass@your-rabbitmq-host:5672/
      - RABBITMQ_QUEUE=whatsapp_events
```

### Running RabbitMQ with WuzAPI

To run RabbitMQ alongside WuzAPI, add this service to your docker-compose.yml:

```yaml
services:
  rabbitmq:
    image: rabbitmq:3-management
    environment:
      RABBITMQ_DEFAULT_USER: wuzapi
      RABBITMQ_DEFAULT_PASS: secure-password
    ports:
      - "5672:5672"    # AMQP port
      - "15672:15672"  # Management UI
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    networks:
      - wuzapi-network

  wuzapi-server:
    environment:
      - RABBITMQ_URL=amqp://wuzapi:secure-password@rabbitmq:5672/
      - RABBITMQ_QUEUE=whatsapp_events
    depends_on:
      rabbitmq:
        condition: service_healthy

volumes:
  rabbitmq_data:
```

## Coolify Deployment

For Coolify deployments, add these environment variables in the Coolify UI:

```
RABBITMQ_URL=amqp://user:pass@your-rabbitmq-host:5672/
RABBITMQ_QUEUE=whatsapp_events
```

## Consuming Messages

Example consumer in Python:

```python
import pika
import json

# Connect to RabbitMQ
connection = pika.BlockingConnection(
    pika.ConnectionParameters('localhost', credentials=pika.PlainCredentials('user', 'pass'))
)
channel = connection.channel()

# Declare queue (idempotent)
channel.queue_declare(queue='whatsapp_events', durable=True)

# Callback function
def callback(ch, method, properties, body):
    event = json.loads(body)
    print(f"Received event: {event['event']} for instance: {event['instanceId']}")
    # Process event here
    ch.basic_ack(delivery_tag=method.delivery_tag)

# Start consuming
channel.basic_consume(queue='whatsapp_events', on_message_callback=callback)
print('Waiting for messages. To exit press CTRL+C')
channel.start_consuming()
```

## Monitoring

Access RabbitMQ Management UI at `http://your-rabbitmq-host:15672` to:
- Monitor queue depth
- View message rates
- Manage queues and exchanges
- Set up alerts

## Troubleshooting

### Connection Refused
- Verify RabbitMQ is running and accessible
- Check firewall rules for port 5672
- Ensure credentials are correct

### Messages Not Publishing
- Check WuzAPI logs for RabbitMQ connection errors
- Verify RABBITMQ_URL is set correctly
- Ensure queue permissions are granted to the user

### High Memory Usage
- Set up RabbitMQ memory limits
- Configure message TTL if messages accumulate
- Implement consumer acknowledgments properly

## Performance Considerations

- RabbitMQ publishing is asynchronous and won't block API responses
- Failed publishes are logged but don't affect WhatsApp operations
- Consider setting up a dead letter exchange for failed messages
- Monitor queue depth to ensure consumers keep up with producers