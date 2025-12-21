# Coolify Installation with Supabase Port Conflict Resolution

## Problem Overview
Installing Coolify on a server that already has Supabase running caused a port conflict on port 8000, which both services try to use by default.

## Initial Issue
- Coolify installation completed but container was unhealthy
- Port 8000 was already occupied by Supabase's Kong API Gateway
- Coolify container couldn't start due to port binding failure

## Solution Steps

### 1. Identify Port Conflict

```bash
# Check what's using port 8000
sudo lsof -i :8000

# Alternative command
ss -tulpn | grep :8000

# Check all containers and their ports
docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

**Result:** Found that `supabase-kong` was using port 8000.

### 2. Stop Supabase Temporarily

```bash
# Stop all Supabase containers to free port 8000
docker ps | grep -E "supabase|realtime-dev" | awk '{print $1}' | xargs docker stop
```

### 3. Fix Docker Compose Configuration

The installed Coolify had incomplete docker-compose files. Fixed by adding missing image definitions:

```bash
cd /data/coolify/source

# Backup the original file
cp docker-compose.prod.yml docker-compose.prod.yml.backup
```

**Created corrected `docker-compose.prod.yml`:**

```yaml
services:
  coolify:
    image: "${REGISTRY_URL:-ghcr.io}/coollabsio/coolify:${LATEST_IMAGE:-latest}"
    volumes:
      - type: bind
        source: /data/coolify/source/.env
        target: /var/www/html/.env
        read_only: true
      - /data/coolify/ssh:/var/www/html/storage/app/ssh
      - /data/coolify/applications:/var/www/html/storage/app/applications
      - /data/coolify/databases:/var/www/html/storage/app/databases
      - /data/coolify/services:/var/www/html/storage/app/services
      - /data/coolify/backups:/var/www/html/storage/app/backups
    environment:
      - APP_ENV=${APP_ENV:-production}
      - PHP_MEMORY_LIMIT=${PHP_MEMORY_LIMIT:-256M}
      - PHP_FPM_PM_CONTROL=${PHP_FPM_PM_CONTROL:-dynamic}
      - PHP_FPM_PM_START_SERVERS=${PHP_FPM_PM_START_SERVERS:-1}
      - PHP_FPM_PM_MIN_SPARE_SERVERS=${PHP_FPM_PM_MIN_SPARE_SERVERS:-1}
      - PHP_FPM_PM_MAX_SPARE_SERVERS=${PHP_FPM_PM_MAX_SPARE_SERVERS:-10}
    env_file:
      - /data/coolify/source/.env
    ports:
      - "${APP_PORT:-8000}:8080"
    expose:
      - "${APP_PORT:-8000}"
    healthcheck:
      test: curl --fail http://127.0.0.1:8080/api/health || exit 1
      interval: 5s
      retries: 10
      timeout: 2s
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      soketi:
        condition: service_healthy
        
  postgres:
    image: postgres:15-alpine
    volumes:
      - coolify-db:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: "${DB_USERNAME}"
      POSTGRES_PASSWORD: "${DB_PASSWORD}"
      POSTGRES_DB: "${DB_DATABASE:-coolify}"
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U ${DB_USERNAME}", "-d", "${DB_DATABASE:-coolify}" ]
      interval: 5s
      retries: 10
      timeout: 2s
    networks:
      default:
        aliases:
          - coolify-db
          - postgres
          
  redis:
    image: redis:7-alpine
    command: redis-server --save 20 1 --loglevel warning --requirepass ${REDIS_PASSWORD}
    environment:
      REDIS_PASSWORD: "${REDIS_PASSWORD}"
    volumes:
      - coolify-redis:/data
    healthcheck:
      test: redis-cli ping
      interval: 5s
      retries: 10
      timeout: 2s
    networks:
      default:
        aliases:
          - coolify-redis
          - redis
          
  soketi:
    image: '${REGISTRY_URL:-ghcr.io}/coollabsio/coolify-realtime:1.0.10'
    ports:
      - "${SOKETI_PORT:-6001}:6001"
      - "6002:6002"
    volumes:
      - /data/coolify/ssh:/var/www/html/storage/app/ssh
    environment:
      APP_NAME: "${APP_NAME:-Coolify}"
      SOKETI_DEBUG: "${SOKETI_DEBUG:-false}"
      SOKETI_DEFAULT_APP_ID: "${PUSHER_APP_ID}"
      SOKETI_DEFAULT_APP_KEY: "${PUSHER_APP_KEY}"
      SOKETI_DEFAULT_APP_SECRET: "${PUSHER_APP_SECRET}"
    healthcheck:
      test: [ "CMD-SHELL", "wget -qO- http://127.0.0.1:6001/ready && wget -qO- http://127.0.0.1:6002/ready || exit 1" ]
      interval: 5s
      retries: 10
      timeout: 2s
      
volumes:
  coolify-db:
    name: coolify-db
  coolify-redis:
    name: coolify-redis
```

**Key changes made:**
1. Added `image: postgres:15-alpine` to postgres service
2. Added `image: redis:7-alpine` to redis service
3. Added network aliases for postgres (`coolify-db`, `postgres`)
4. Added network aliases for redis (`coolify-redis`, `redis`)

### 4. Fix Database Authentication Issue

After fixing the compose file, encountered database authentication errors. This was resolved by recreating the database volume:

```bash
cd /data/coolify/source

# Stop all containers
docker compose -f docker-compose.prod.yml down

# Remove the database volume to start fresh
docker volume rm coolify-db

# Start everything with clean database
docker compose -f docker-compose.prod.yml up -d

# Wait for services to start
sleep 40

# Check logs
docker logs source-coolify-1 --tail 30
```

### 5. Verify Installation

```bash
# Check all containers are running and healthy
docker ps | grep coolify

# View logs if needed
docker logs source-coolify-1 --tail 50
```

**Expected output:**
- `source-coolify-1` - running and healthy
- `source-postgres-1` - running and healthy
- `source-redis-1` - running and healthy
- `source-soketi-1` - running and healthy

### 6. Access Coolify

Coolify is now accessible at: `http://your-server-ip:8000`

## Setting Up Domain Access with Nginx

To access Coolify via a custom domain using Nginx as a reverse proxy:

### 1. Install Nginx (if not already installed)

```bash
# Check if nginx is installed
nginx -v

# If not installed:
sudo apt update
sudo apt install nginx -y
```

### 2. Create Nginx Configuration

```bash
sudo nano /etc/nginx/sites-available/coolify
```

Add this configuration (replace `coolify.yourdomain.com` with your actual domain):

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name coolify.yourdomain.com;
    
    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Timeouts
        proxy_connect_timeout 600;
        proxy_send_timeout 600;
        proxy_read_timeout 600;
        send_timeout 600;
    }
}
```

### 3. Enable the Site

```bash
# Create symbolic link
sudo ln -s /etc/nginx/sites-available/coolify /etc/nginx/sites-enabled/

# If link already exists, remove and recreate:
sudo rm /etc/nginx/sites-enabled/coolify
sudo ln -s /etc/nginx/sites-available/coolify /etc/nginx/sites-enabled/

# Test nginx configuration
sudo nginx -t

# Reload nginx
sudo systemctl reload nginx

# Check nginx status
sudo systemctl status nginx
```

### 4. Configure DNS

Point your domain (A record) to your server's IP address.

Verify DNS resolution:
```bash
nslookup coolify.yourdomain.com
# Or
dig coolify.yourdomain.com +short
```

### 5. Add SSL with Let's Encrypt

```bash
# Install certbot
sudo apt install certbot python3-certbot-nginx -y

# Get SSL certificate (replace with your domain)
sudo certbot --nginx -d coolify.yourdomain.com
```

Certbot will automatically:
- Obtain SSL certificate from Let's Encrypt
- Modify Nginx config for HTTPS
- Set up automatic certificate renewal

### 6. Access Coolify via Domain

- **HTTP:** `http://coolify.yourdomain.com`
- **HTTPS:** `https://coolify.yourdomain.com` (after SSL setup)

## Running Both Services Simultaneously

To run both Coolify and Supabase together, configure Supabase to use a different port:

### Find Supabase Installation

```bash
find / -name "docker-compose.yml" -path "*supabase*" 2>/dev/null
```

### Change Supabase Kong Port

Edit the Supabase `docker-compose.yml`:

```bash
cd /path/to/supabase

# Backup first
cp docker-compose.yml docker-compose.yml.backup

# Edit the file
nano docker-compose.yml
```

Find the Kong service and change:
```yaml
kong:
  ports:
    - "8000:8000"  # Change this line
    - "8443:8443"
```

To:
```yaml
kong:
  ports:
    - "8080:8000"  # Changed to 8080
    - "8443:8443"
```

### Restart Supabase

```bash
docker compose down
docker compose up -d
```

### Access Both Services

- **Coolify:** `http://your-server-ip:8000` or `https://coolify.yourdomain.com`
- **Supabase:** `http://your-server-ip:8080`

## Auto-Start on Server Reboot

### Coolify
Coolify will automatically start on reboot because it was created with docker compose (default restart policy).

### Supabase
After changing the port and starting Supabase with `docker compose up -d`, it will also auto-start on reboot.

Verify restart policies:
```bash
# Check restart policies
docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.RestartCount}}"

# Or inspect specific container
docker inspect source-coolify-1 | grep -A 3 RestartPolicy
```

## Troubleshooting

### Container Can't Resolve Hostnames

If you see errors like "could not translate host name coolify-db":

```bash
# Check network configuration
docker inspect <container-name> | grep -A 10 Networks

# Ensure containers have proper network aliases in docker-compose.yml
```

### Port Already in Use

```bash
# Find what's using a port
sudo lsof -i :8000

# Check docker-proxy processes
docker ps -a --format "{{.ID}} {{.Names}} {{.Ports}}" | grep 8000
```

### Database Authentication Failed

```bash
# Verify credentials match
grep DB_USERNAME /data/coolify/source/.env
grep DB_PASSWORD /data/coolify/source/.env
docker exec source-postgres-1 env | grep POSTGRES

# If mismatched, recreate database volume
docker compose down
docker volume rm coolify-db
docker compose up -d
```

### Nginx Configuration Issues

```bash
# Test nginx configuration
sudo nginx -t

# Check nginx logs
sudo tail -f /var/log/nginx/error.log

# Restart nginx
sudo systemctl restart nginx
```

### SSL Certificate Issues

```bash
# Check certificate status
sudo certbot certificates

# Renew certificates manually
sudo certbot renew

# Test renewal process
sudo certbot renew --dry-run
```

## Key Learnings

1. **Port conflicts** - Always check for existing services using the same port before installation
2. **Network aliases** - Docker services need proper DNS aliases to communicate by hostname
3. **Environment variables** - Ensure all required environment variables are set before starting containers
4. **Database persistence** - When credentials don't match, sometimes it's easier to recreate the volume than troubleshoot
5. **Docker Compose** - Always use compose files to manage multi-container applications for proper networking
6. **Reverse proxies** - Using Nginx provides flexibility for SSL, domain mapping, and running multiple services
7. **Auto-start** - Services managed with docker compose will auto-start on reboot by default

## References

- Coolify Documentation: https://coolify.io/docs
- Docker Compose Networking: https://docs.docker.com/compose/networking/
- Supabase Self-Hosting: https://supabase.com/docs/guides/self-hosting
- Nginx Reverse Proxy: https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/
- Let's Encrypt: https://letsencrypt.org/getting-started/
