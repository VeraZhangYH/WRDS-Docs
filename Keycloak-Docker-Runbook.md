# Keycloak Docker Deployment Runbook

## Overview
This runbook documents the Docker-based Keycloak deployment for our single sign-on (SSO) system. The setup uses Docker Compose to manage both Keycloak and PostgreSQL containers.

## Environment Details
- **Host Server**: 192.168.168.118
- **Keycloak Version**: 22.0.5
- **Database**: PostgreSQL 14
- **Container Names**: 
  - Keycloak: `keycloak-new`
  - PostgreSQL: `keycloak-postgres`

## Docker Configuration
The deployment is managed through the following `docker-compose.yml`:

```yaml
version: '3'

services:
  postgres:
    image: postgres:14
    container_name: keycloak-postgres
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: yanci
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - keycloak-network

  keycloak:
    image: quay.io/keycloak/keycloak:22.0.5
    container_name: keycloak-new
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres:5432/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: yanci
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: yanci
      KC_HTTP_ENABLED: "true"
      KC_HOSTNAME: 192.168.168.118
      KC_HTTPS_REQUIRED: "none"
      KC_HOSTNAME_STRICT_HTTPS: "false"
      KC_PROXY: edge
    command: start-dev
    ports:
      - "8080:8080"
    # Remove certificate mounts if just using HTTP
    # volumes:
    #   - ../myserver.crt:/opt/keycloak/conf/server.crt.pem
    #   - ../myserver.key:/opt/keycloak/conf/server.key.pem
    depends_on:
      - postgres
    networks:
      - keycloak-network

volumes:
  postgres_data:

networks:
  keycloak-network:
    driver: bridge
```

## Basic Operations

### Starting Keycloak
```bash
# Navigate to the directory containing docker-compose.yml
cd ~/keycloak-docker

# Start in detached mode
docker-compose up -d
```

### Stopping Keycloak
```bash
# Navigate to the directory containing docker-compose.yml
cd ~/keycloak-docker

# Option 1: Stop containers but keep them
docker-compose stop

# Option 2: Stop and remove containers (data persists in volumes)
docker-compose down
```

### Restarting Keycloak
```bash
# Restart all services
docker-compose restart

# Restart only Keycloak
docker-compose restart keycloak
```

### Checking Status
```bash
# View status of all containers
docker-compose ps

# View detailed status
docker ps | grep keycloak
```

## Logs and Monitoring

### Viewing Logs
```bash
# View Keycloak logs
docker logs keycloak-new

# Follow logs in real-time
docker logs -f keycloak-new

# View PostgreSQL logs
docker logs keycloak-postgres
```

### Monitoring Performance
```bash
# Check resource usage
docker stats keycloak-new keycloak-postgres
```

## Configuration Management

### Modifying Environment Variables
1. Edit the `docker-compose.yml` file
2. Update the required environment variables
3. Restart the containers:
   ```bash
   docker-compose down
   docker-compose up -d
   ```

### Accessing Admin Console
1. Open a web browser
2. Navigate to `http://192.168.168.118:8080/admin/`
3. Login with:
   - Username: admin
   - Password: yanci

## Database Management

### Creating Database Backup
```bash
# Create a backup with timestamp
docker exec keycloak-postgres pg_dump -U keycloak keycloak > keycloak_backup_$(date +%Y%m%d).sql
```

### Restoring from Backup
```bash
# Restore from backup file
cat backup_file.sql | docker exec -i keycloak-postgres psql -U keycloak -d keycloak
```

### Connecting to Database
```bash
# Connect to PostgreSQL
docker exec -it keycloak-postgres psql -U keycloak -d keycloak
```

## Maintenance Tasks

### Updating Keycloak Version
1. Update the image version in `docker-compose.yml`
2. Pull the new image and restart:
   ```bash
   docker-compose pull
   docker-compose up -d
   ```

### Managing Volumes
```bash
# List volumes
docker volume ls | grep keycloak

# Backup a volume
docker run --rm -v keycloak-docker_postgres_data:/source -v $(pwd):/backup alpine tar -czvf /backup/postgres_data_backup.tar.gz /source
```

## Troubleshooting

### Container Won't Start
```bash
# Check for errors in logs
docker logs keycloak-new

# Check container details
docker inspect keycloak-new

# Verify network connectivity
docker network inspect keycloak-docker_keycloak-network
```

### Database Connection Issues
```bash
# Test connection from Keycloak to PostgreSQL
docker exec -it keycloak-new bash -c 'nc -zv postgres 5432'

# Check PostgreSQL logs
docker logs keycloak-postgres
```

### Common Error Resolutions
1. **Port Conflicts**: Verify port 8080 is not in use by another service
   ```bash
   netstat -tulpn | grep 8080
   ```

2. **Memory Issues**: Ensure host has sufficient resources
   ```bash
   free -h
   ```

3. **Permission Problems**: Check volume permissions
   ```bash
   ls -la /var/lib/docker/volumes/
   ```

## Security Considerations

### SSL Configuration
For production environments, modify the `docker-compose.yml` to enable HTTPS:
```yaml
keycloak:
  environment:
    KC_HTTPS_REQUIRED: "all"
  volumes:
    - ./certs/server.crt:/opt/keycloak/conf/server.crt.pem
    - ./certs/server.key:/opt/keycloak/conf/server.key.pem
```

### Network Security
1. Configure host firewall to limit access to port 8080
2. Consider using a reverse proxy for TLS termination

### Password Management
1. Store database and admin passwords in a secure password manager
2. Consider using Docker secrets for sensitive information

## Reference Information

### Useful Commands
```bash
# Connect to Keycloak container shell
docker exec -it keycloak-new bash

# View Docker Compose configuration
docker-compose config

# Check Docker disk usage
docker system df
```

### Documentation Links
- Keycloak Documentation: https://www.keycloak.org/documentation
- Docker Compose Reference: https://docs.docker.com/compose/reference/
- PostgreSQL Documentation: https://www.postgresql.org/docs/14/index.html

---

*This runbook was last updated on: [March/7/2025]*
