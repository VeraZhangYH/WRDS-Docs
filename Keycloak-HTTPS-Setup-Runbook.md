# Keycloak HTTPS Setup Runbook: Nginx Reverse Proxy Method

## Problem Statement

Our team needed to secure a Keycloak authentication server that was initially deployed using Docker and exposed via HTTP on port 8080. For production use, we needed to:

1. Implement HTTPS to encrypt all traffic
2. Maintain the current port (8080) for backward compatibility
3. Ensure proper security practices for an identity provider
4. Set up a solution that would be maintainable and scalable

## Solution Comparison

We evaluated two approaches for implementing HTTPS:

### Direct Port Mapping Method
Configuring Keycloak container to directly handle HTTPS and mapping the container's HTTPS port to host's 8080 port.

### Reverse Proxy Method (Selected)
Using Nginx as a reverse proxy to handle HTTPS while proxying requests to Keycloak running on HTTP internally.

## Why We Chose the Reverse Proxy Method

The reverse proxy approach offers several advantages:

- **Security Layering**: Adds an additional security barrier between internet and Keycloak
- **Advanced Features**: Provides traffic management, rate limiting, and request filtering
- **SSL Termination**: Offloads SSL processing, reducing load on Keycloak
- **Flexibility**: Allows easy modification of security headers and redirect rules
- **Future-Proofing**: Enables simpler addition of WAF, load balancing, or other proxies later
- **Better Logging**: More detailed access logging and monitoring capabilities
- **Industry Standard**: Follows best practices for production deployment of identity services

## Implementation Steps

### Step 1: Install Nginx

```bash
# Update package list and install Nginx
sudo apt update
sudo apt install nginx -y
```

### Step 2: Create SSL Directory and Copy Certificates

```bash
# Create directory for SSL certificates
sudo mkdir -p /etc/nginx/ssl

# Copy existing certificates
sudo cp /home/yanci/myserver.crt /etc/nginx/ssl/
sudo cp /home/yanci/myserver.key /etc/nginx/ssl/

# Set proper permissions
sudo chmod 644 /etc/nginx/ssl/myserver.crt
sudo chmod 600 /etc/nginx/ssl/myserver.key
```

### Step 3: Configure Nginx for Keycloak

Created a new Nginx configuration file:

```bash
# Create Nginx configuration file
sudo nano /etc/nginx/sites-available/keycloak.conf
```

With the following content:

```nginx
server {
    listen 8080 ssl;
    server_name 192.168.168.118;

    # SSL configuration
    ssl_certificate /etc/nginx/ssl/myserver.crt;
    ssl_certificate_key /etc/nginx/ssl/myserver.key;

    # Security settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:D>
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # Additional security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Frame-Options SAMEORIGIN;

    # Proxy settings for Keycloak - CHANGED from localhost to 127.0.0.1
    location / {
        proxy_pass http://172.18.0.3:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;

        # WebSocket support (for Keycloak admin console)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Increased timeouts to prevent connection errors
        proxy_connect_timeout 120;
        proxy_send_timeout 120;
        proxy_read_timeout 120;
    }

    # Optimize file access for static content - CHANGED from localhost to 127.0.0.1
    location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
        proxy_pass http://172.18.0.3:8081;
        proxy_set_header Host $host:$server_port;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host $host:$server_port;
        proxy_set_header X-Forwarded-Port $server_port;
        proxy_cache_valid 200 1d;
        access_log off;
        expires 30d;
        add_header Pragma public;
        add_header Cache-Control "public";
    }
}

```

### Step 4: Update Docker Compose for Keycloak

Updated the docker-compose.yml file to:

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
     * # Database configuration*
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres:5432/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: yanci
      
     * # Admin credentials*
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: yanci
      
     * # Frontend URL configuration (critical for redirects)*
      KC_HOSTNAME_URL: https://192.168.168.118:8080
      KC_HOSTNAME_ADMIN_URL: https://192.168.168.118:8080

     * # HTTP configuration - updated for reverse proxy*
      KC_HTTP_ENABLED: "true"
      KC_HTTP_PORT: "8081" * # Changed to 8081 for Nginx proxy*
      
     * # Proxy settings*
      KC_PROXY: edge
      KC_PROXY_ADDRESS_FORWARDING: "true"
      
     * # HTTPS settings*
      KC_HTTPS_REQUIRED: "none"
      KC_HOSTNAME_STRICT_HTTPS: "false"
      KC_HOSTNAME_STRICT: "false"
    
   * # Changed to production mode*
    command: start
    
   * # Updated port mapping*
    ports:
      - "8081:8081"
      
   * # No need for certificate mounts as Nginx will handle SSL*
   * # volumes:*
   * #   - ../myserver.crt:/opt/keycloak/conf/server.crt.pem*
   * #   - ../myserver.key:/opt/keycloak/conf/server.key.pem*
    
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

Key changes:
- Changed port from 8080 to 8081
- Set KC_HTTP_PORT to 8081
- Changed command from start-dev to start for production mode
- Added proxy and hostname configuration

### Step 5: Enable the Nginx Configuration and Test

```bash
# Enable the site
sudo ln -s /etc/nginx/sites-available/keycloak.conf /etc/nginx/sites-enabled/

# Test Nginx configuration
sudo nginx -t

# Restart Nginx
sudo systemctl restart nginx
```

### Step 6: Restart Keycloak with New Configuration

```bash
# Navigate to docker-compose directory
cd ~/keycloak-docker

# Stop existing containers
docker-compose down

# Start with new configuration
docker-compose up -d
```

### Step 7: Check Services and Verify

```bash
# Check Nginx status
sudo systemctl status nginx

# Check Docker containers
docker ps

# Check Keycloak logs
docker logs keycloak-new
```

### Step 8: Test the HTTPS Connection

Verified access to Keycloak via HTTPS by accessing:
```
https://192.168.168.118:8080/admin/master/console/
```

Successfully logged in with admin credentials, confirming the secure connection is working properly.

## Troubleshooting

If issues are encountered, these commands can help diagnose problems:

### Nginx Issues:
```bash
# Check Nginx error logs
sudo tail -f /var/log/nginx/error.log

# Check access logs
sudo tail -f /var/log/nginx/access.log
```

### Keycloak Issues:
```bash
# Check Keycloak logs
docker logs -f keycloak-new
```

### Connection Issues:
```bash
# Check if ports are open
sudo netstat -tuln | grep 8080
sudo netstat -tuln | grep 8081

# Check if Nginx is listening on 8080
sudo lsof -i :8080
```

## Maintenance Considerations

1. **Certificate Renewal**: SSL certificates will need to be renewed before expiration
2. **Security Updates**: Regularly update Nginx, Docker, and Keycloak for security patches
3. **Backup Strategy**: Implement regular backups of PostgreSQL data and Keycloak configuration
4. **Monitoring**: Consider setting up monitoring for service availability and certificate expiration
5. **Automatic Restart**: Configure services to restart automatically after server reboots

## Conclusion

By implementing HTTPS with Nginx as a reverse proxy, we've successfully secured our Keycloak instance while maintaining the original port (8080) for consistent access. This approach follows industry best practices for deploying identity providers in production environments and offers flexibility for future enhancements.
