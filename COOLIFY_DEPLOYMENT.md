# Deploying WuzAPI to Coolify

This guide walks you through deploying WuzAPI to Coolify using Docker Compose.

## Prerequisites

- A Coolify instance up and running
- A GitHub/GitLab repository containing your WuzAPI code
- Basic understanding of Docker and environment variables

## Step 1: Prepare Your Repository

1. Create a production-ready `docker-compose.coolify.yml` file in your repository root:

```yaml
services:
  wuzapi-server:
    build:
      context: .
      dockerfile: Dockerfile
    expose:
      - "8080"
    environment:
      - WUZAPI_ADMIN_TOKEN=${WUZAPI_ADMIN_TOKEN}
      - DB_USER=${DB_USER:-wuzapi}
      - DB_PASSWORD=${DB_PASSWORD}
      - DB_NAME=${DB_NAME:-wuzapi}
      - DB_HOST=db
      - DB_PORT=5432
      - TZ=${TZ:-UTC}
      - WEBHOOK_FORMAT=${WEBHOOK_FORMAT:-json}
      - SESSION_DEVICE_NAME=${SESSION_DEVICE_NAME:-WuzAPI}
    depends_on:
      db:
        condition: service_healthy
    labels:
      - "coolify.managed=true"
      - "coolify.type=application"
      - "coolify.name=wuzapi"
      - "coolify.port=8080"
    restart: unless-stopped

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: ${DB_USER:-wuzapi}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME:-wuzapi}
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-wuzapi}"]
      interval: 5s
      timeout: 5s
      retries: 5
    restart: unless-stopped

volumes:
  db_data:
    driver: local
```

2. Commit and push this file to your repository.

## Step 2: Configure Coolify

1. **Create a New Resource**
   - Log into your Coolify dashboard
   - Navigate to your project
   - Click "New Resource"
   - Select "Docker Compose"

2. **Connect Your Repository**
   - Choose your Git provider (GitHub/GitLab/etc.)
   - Provide repository URL
   - Configure deploy key if needed
   - Select branch (usually `main` or `master`)

3. **Configure Build Settings**
   - Build Pack: Docker Compose
   - Compose file path: `docker-compose.coolify.yml`
   - Base directory: `/` (or your project root)

## Step 3: Set Environment Variables

In Coolify's environment variables section, add:

```bash
# Required - Generate a secure admin token
WUZAPI_ADMIN_TOKEN=your-secure-admin-token-here

# Database credentials
DB_USER=wuzapi
DB_PASSWORD=your-secure-database-password
DB_NAME=wuzapi

# Optional configurations
TZ=America/New_York
WEBHOOK_FORMAT=json
SESSION_DEVICE_NAME=WuzAPI-Production

# Optional: RabbitMQ (if using)
# RABBITMQ_URL=amqp://user:pass@rabbitmq:5672/

# Optional: S3 (if using)
# S3_ACCESS_KEY_ID=your-access-key
# S3_SECRET_ACCESS_KEY=your-secret-key
# S3_BUCKET_NAME=your-bucket-name
# S3_REGION=us-east-1
```

### Generating a Secure Admin Token

```bash
# Option 1: Using openssl
openssl rand -hex 32

# Option 2: Using uuidgen
uuidgen | tr -d '-'

# Option 3: Using pwgen
pwgen -s 32 1
```

## Step 4: Configure Domain and SSL

1. In Coolify, navigate to your application settings
2. Set your domain (e.g., `wuzapi.yourdomain.com`)
3. Enable "Force HTTPS"
4. Coolify will automatically provision SSL certificates via Let's Encrypt

## Step 5: Deploy

1. Click the "Deploy" button in Coolify
2. Monitor the deployment logs
3. Wait for the build and deployment to complete

## Step 6: Verify Deployment

Once deployed, verify your installation:

```bash
# Check API status
curl https://wuzapi.yourdomain.com/admin/users \
  -H "Authorization: your-admin-token"

# Access Swagger UI
# Navigate to: https://wuzapi.yourdomain.com/api
```

## Production Considerations

### Database Backup

For production, consider using Coolify's database service or an external managed database:

1. **Option 1: Coolify Database Service**
   - Create a PostgreSQL database in Coolify
   - Update your compose file to use the external database
   - Remove the `db` service from docker-compose

2. **Option 2: External Database (Recommended)**
   - Use a managed database service (AWS RDS, DigitalOcean, etc.)
   - Update environment variables with external database credentials

### Persistent Storage

If storing media locally (not using S3), add a volume for media storage:

```yaml
services:
  wuzapi-server:
    volumes:
      - media_data:/app/media

volumes:
  media_data:
    driver: local
```

### Resource Limits

Add resource constraints for production:

```yaml
services:
  wuzapi-server:
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 512M
```

### Health Checks

Add health check for the API:

```yaml
services:
  wuzapi-server:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

## Monitoring and Logs

1. **View Logs**: In Coolify, click on your application and navigate to "Logs"
2. **Real-time Logs**: Use the "Follow Logs" option
3. **Container Stats**: Monitor CPU and memory usage in the Coolify dashboard

## Updating Your Application

1. Push changes to your Git repository
2. In Coolify, click "Redeploy" or enable auto-deploy
3. Coolify will pull the latest changes and redeploy

## Troubleshooting

### Common Issues

1. **Database Connection Failed**
   - Verify DB_HOST is set to `db` (service name)
   - Check database credentials match
   - Ensure database service is healthy

2. **Port Already in Use**
   - Coolify handles port mapping automatically
   - Use `expose` instead of `ports` in compose file

3. **Build Failures**
   - Check Dockerfile syntax
   - Verify all required files are in repository
   - Review build logs in Coolify

### Debug Commands

Access your container via Coolify's terminal:

```bash
# Check database connection
pg_isready -h db -U wuzapi

# Test API endpoint
curl http://localhost:8080/health

# View environment variables
env | grep -E "WUZAPI|DB_"
```

## Security Best Practices

1. **Strong Admin Token**: Use a cryptographically secure token
2. **Database Password**: Use strong, unique passwords
3. **Network Isolation**: Coolify automatically creates isolated networks
4. **Regular Updates**: Keep base images and dependencies updated
5. **Limit Exposed Ports**: Only expose necessary services
6. **Use HTTPS**: Always enable SSL in production

## Additional Resources

- [Coolify Documentation](https://coolify.io/docs)
- [WuzAPI Documentation](./API.md)
- [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)

## Support

For WuzAPI issues: Check the project's GitHub issues
For Coolify issues: Visit the Coolify community forums or Discord