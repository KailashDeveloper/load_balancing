# MySQL Load Balancer with HAProxy - Server Integration Guide

## 📋 Overview

This documentation provides step-by-step instructions for integrating new servers into an existing MySQL load balancing setup with HAProxy, featuring automatic CPU-based failover capabilities.

## 🏗️ Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   PHP App       │    │    HAProxy      │    │  MySQL Master   │
│   Port: 9007    │────│   Port: 3309    │────│   Port: 3307    │
└─────────────────┘    │   Stats: 8080   │    └─────────────────┘
                       └─────────────────┘              │
                               │                        │
                               │              ┌─────────────────┐
                               └──────────────│  MySQL Slave    │
                                              │   Port: 3308    │
                                              └─────────────────┘
```

## 📦 Components

- **HAProxy**: Load balancer with health checks and automatic failover
- **MySQL Master**: Primary database server
- **MySQL Slave**: Backup database server with replication
- **PHP Application**: Web application connecting through HAProxy
- **CPU Monitor**: Automatic failover based on CPU usage
- **phpMyAdmin**: Database management interface

## 🚀 Quick Start - New Server Integration

### Prerequisites

```bash
# Required software
- Docker & Docker Compose
- Git
- Basic Linux commands knowledge
```

### Step 1: Clone Repository and Setup

```bash
# Clone the repository to new server
git clone <your-repo-url> /www/docker/mysql-loadbalancer
cd /www/docker/mysql-loadbalancer

# Create necessary directories
mkdir -p haproxy php-app/src logs

# Set proper permissions
chmod -R 755 .
```

### Step 2: Environment Configuration

Create `.env` file:

```bash
cat > .env << 'EOF'
# MySQL Configuration
MYSQL_ROOT_PASSWORD=rootpassword123
MYSQL_DATABASE=testdb
MYSQL_USER=dbuser
MYSQL_PASSWORD=dbpassword123

# HAProxy Configuration
HAPROXY_STATS_USER=admin
HAPROXY_STATS_PASSWORD=admin123

# Server Configuration
SERVER_IP=YOUR_SERVER_IP
PHP_PORT=9007
HAPROXY_STATS_PORT=8080
MYSQL_LB_PORT=3309
MYSQL_MASTER_PORT=3307
MYSQL_SLAVE_PORT=3308
PHPMYADMIN_PORT=9008
PHPMYADMIN_SLAVE_PORT=9009
EOF
```

### Step 3: Docker Compose Configuration

Create `docker-compose.yml`:

```yaml
services:
  # HAProxy Load Balancer
  haproxy:
    image: haproxy:2.8
    container_name: mysql_loadbalancer
    ports:
      - "${MYSQL_LB_PORT:-3309}:3306"
      - "${HAPROXY_STATS_PORT:-8080}:8080"
    volumes:
      - ./haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
      - ./logs:/var/log/haproxy
    depends_on:
      - mysql1
      - mysql2
    restart: unless-stopped
    networks:
      - mysql_network

  # MySQL Master
  mysql1:
    image: mysql:8.0
    container_name: mysql_master
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    command: --server-id=1 --log-bin=mysql-bin --binlog-format=ROW --max_allowed_packet=1073741824
    ports:
      - "${MYSQL_MASTER_PORT:-3307}:3306"
    volumes:
      - mysql1_data:/var/lib/mysql
      - ./mysql/master:/docker-entrypoint-initdb.d
    restart: unless-stopped
    networks:
      - mysql_network

  # MySQL Slave
  mysql2:
    image: mysql:8.0
    container_name: mysql_slave
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    command: --server-id=2 --log-bin=mysql-bin --binlog-format=ROW --max_allowed_packet=1073741824
    ports:
      - "${MYSQL_SLAVE_PORT:-3308}:3306"
    volumes:
      - mysql2_data:/var/lib/mysql
      - ./mysql/slave:/docker-entrypoint-initdb.d
    restart: unless-stopped
    networks:
      - mysql_network

  # PHP Application
  php-app:
    build: ./php-app
    container_name: php_application
    environment:
      DB_HOST: haproxy
      DB_PORT: 3306
      DB_NAME: ${MYSQL_DATABASE}
      DB_USER: root
      DB_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    ports:
      - "${PHP_PORT:-9007}:80"
    volumes:
      - ./php-app/src:/var/www/html
    depends_on:
      - haproxy
    restart: unless-stopped
    networks:
      - mysql_network

  # phpMyAdmin for Load Balancer
  phpmyadmin:
    image: phpmyadmin/phpmyadmin:latest
    container_name: phpmyadmin_interface
    environment:
      PMA_HOST: haproxy
      PMA_PORT: 3306
      PMA_USER: root
      PMA_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      UPLOAD_LIMIT: 1G
      MEMORY_LIMIT: 2G
      MAX_EXECUTION_TIME: 3600
    ports:
      - "${PHPMYADMIN_PORT:-9008}:80"
    depends_on:
      - haproxy
    restart: unless-stopped
    networks:
      - mysql_network

  # phpMyAdmin for Slave
  phpmyadmin_slave:
    image: phpmyadmin/phpmyadmin:latest
    container_name: phpmyadmin_slave
    environment:
      PMA_HOST: mysql2
      PMA_PORT: 3306
      PMA_USER: root
      PMA_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      UPLOAD_LIMIT: 1G
      MEMORY_LIMIT: 2G
      MAX_EXECUTION_TIME: 3600
    ports:
      - "${PHPMYADMIN_SLAVE_PORT:-9009}:80"
    depends_on:
      - mysql2
    restart: unless-stopped
    networks:
      - mysql_network

volumes:
  mysql1_data:
  mysql2_data:

networks:
  mysql_network:
    driver: bridge
```

### Step 4: HAProxy Configuration

Create `haproxy/haproxy.cfg`:

```bash
mkdir -p haproxy
cat > haproxy/haproxy.cfg << 'EOF'
global
    daemon
    log stdout local0
    stats socket /tmp/haproxy.sock mode 666 level admin
    stats timeout 30s

defaults
    mode tcp
    log global
    option tcplog
    option dontlognull
    retries 2
    timeout connect 2000ms
    timeout client 30000ms
    timeout server 30000ms

# HAProxy Statistics Interface
listen stats
    bind *:8080
    mode http
    stats enable
    stats uri /stats
    stats refresh 5s
    stats show-legends
    stats show-node
    stats admin if TRUE
    stats auth admin:admin123
    stats realm HAProxy\ MySQL\ Load\ Balancer

# MySQL Cluster with CPU-based Failover
listen mysql-cluster
    bind *:3306
    mode tcp
    balance first
    option tcplog
    
    # Health checks for automatic failover
    default-server check inter 500ms rise 1 fall 1 fastinter 200ms downinter 1s
    
    # Master server - primary
    server mysql-master mysql1:3306 weight 100 maxconn 100
    
    # Slave server - automatic backup
    server mysql-slave mysql2:3306 weight 90 maxconn 100 backup
EOF
```

### Step 5: PHP Application Setup

Create PHP application structure:

```bash
mkdir -p php-app/src
cat > php-app/Dockerfile << 'EOF'
FROM php:8.2-apache

# Install required extensions
RUN docker-php-ext-install mysqli pdo pdo_mysql

# Enable Apache modules
RUN a2enmod rewrite

# Set working directory
WORKDIR /var/www/html

# Copy application files
COPY src/ /var/www/html/

# Set permissions
RUN chown -R www-data:www-data /var/www/html
RUN chmod -R 755 /var/www/html
EOF

# Create sample PHP application
cat > php-app/src/index.php << 'EOF'
<?php
echo "<h1>MySQL Load Balancer Test</h1>";
echo "<p>Server Time: " . date("Y-m-d H:i:s") . "</p>";

// Database connection test
try {
    $pdo = new PDO(
        "mysql:host=" . ($_ENV['DB_HOST'] ?? 'haproxy') . ";port=" . ($_ENV['DB_PORT'] ?? '3306') . ";dbname=" . ($_ENV['DB_NAME'] ?? 'testdb'),
        $_ENV['DB_USER'] ?? 'root',
        $_ENV['DB_PASSWORD'] ?? 'rootpassword123'
    );
    
    $stmt = $pdo->query("SELECT @@server_id as server_id, @@hostname as hostname, NOW() as current_time");
    $result = $stmt->fetch();
    
    echo "<h2>Database Connection: ✅ Success</h2>";
    echo "<p>Server ID: " . $result['server_id'] . "</p>";
    echo "<p>Hostname: " . $result['hostname'] . "</p>";
    echo "<p>DB Time: " . $result['current_time'] . "</p>";
    
} catch (Exception $e) {
    echo "<h2>Database Connection: ❌ Failed</h2>";
    echo "<p>Error: " . $e->getMessage() . "</p>";
}
?>
EOF

# Create events.php for testing
cat > php-app/src/events.php << 'EOF'
<?php
$page = $_GET['page'] ?? 1;

echo "<h1>Events Page $page</h1>";
echo "<p>Generated at: " . date("Y-m-d H:i:s") . "</p>";

try {
    $pdo = new PDO(
        "mysql:host=" . ($_ENV['DB_HOST'] ?? 'haproxy') . ";port=" . ($_ENV['DB_PORT'] ?? '3306') . ";dbname=" . ($_ENV['DB_NAME'] ?? 'testdb'),
        $_ENV['DB_USER'] ?? 'root',
        $_ENV['DB_PASSWORD'] ?? 'rootpassword123'
    );
    
    $stmt = $pdo->query("SELECT @@server_id as server_id, @@hostname as hostname");
    $result = $stmt->fetch();
    
    echo "<p>Using MySQL Server: " . $result['server_id'] . " (" . $result['hostname'] . ")</p>";
    echo "<p>Page: $page</p>";
    
    // Simulate some processing time
    usleep(100000); // 100ms
    
} catch (Exception $e) {
    echo "<p>Database Error: " . $e->getMessage() . "</p>";
}
?>
EOF
```

### Step 6: Deployment Scripts

Create deployment and monitoring scripts:

```bash
# Main deployment script
cat > deploy.sh << 'EOF'
#!/bin/bash

echo "=== MySQL Load Balancer Deployment ==="
echo "Starting deployment at: $(date)"

# Stop existing containers
echo "Stopping existing containers..."
docker compose down

# Pull latest images
echo "Pulling latest Docker images..."
docker compose pull

# Build custom images
echo "Building PHP application..."
docker compose build

# Start services
echo "Starting services..."
docker compose up -d

# Wait for services to start
echo "Waiting for services to start..."
sleep 30

# Verify deployment
echo "Verifying deployment..."
./verify_deployment.sh

echo "Deployment completed at: $(date)"
EOF

# Verification script
cat > verify_deployment.sh << 'EOF'
#!/bin/bash

echo "=== Deployment Verification ==="

# Check container status
echo "Container Status:"
docker compose ps

echo ""
echo "Service Health Checks:"

# Test HAProxy stats
if curl -s -u admin:admin123 http://localhost:8080/stats > /dev/null; then
    echo "✅ HAProxy Stats: OK"
else
    echo "❌ HAProxy Stats: FAILED"
fi

# Test PHP application
if curl -s http://localhost:9007/ > /dev/null; then
    echo "✅ PHP Application: OK"
else
    echo "❌ PHP Application: FAILED"
fi

# Test MySQL connection
if mysql -h localhost -P 3309 -uroot -prootpassword123 -e "SELECT 1" > /dev/null 2>&1; then
    echo "✅ MySQL Load Balancer: OK"
else
    echo "❌ MySQL Load Balancer: FAILED"
fi

# Test phpMyAdmin
if curl -s http://localhost:9008/ > /dev/null; then
    echo "✅ phpMyAdmin: OK"
else
    echo "❌ phpMyAdmin: FAILED"
fi

echo ""
echo "Access URLs:"
echo "- PHP Application: http://YOUR_SERVER_IP:9007/"
echo "- HAProxy Stats: http://YOUR_SERVER_IP:8080/stats (admin/admin123)"
echo "- phpMyAdmin: http://YOUR_SERVER_IP:9008/"
echo "- phpMyAdmin Slave: http://YOUR_SERVER_IP:9009/"
EOF

chmod +x deploy.sh verify_deployment.sh
```

### Step 7: CPU Monitoring and Failover

Create CPU monitoring script:

```bash
cat > cpu_monitor.sh << 'EOF'
#!/bin/bash

# CPU-based MySQL Failover Monitor
CPU_THRESHOLD=85
CHECK_INTERVAL=10
LOG_FILE="/var/log/mysql_cpu_monitor.log"

echo "=== MySQL CPU Monitor Started ===" | tee -a $LOG_FILE
echo "CPU Threshold: ${CPU_THRESHOLD}%" | tee -a $LOG_FILE
echo "Check Interval: ${CHECK_INTERVAL}s" | tee -a $LOG_FILE
echo "Started: $(date)" | tee -a $LOG_FILE

# Function to get CPU usage
get_cpu_usage() {
    top -bn1 | grep "Cpu(s)" | awk '{print $2}' | sed 's/%us,//' | cut -d'.' -f1
}

# Function to switch servers
switch_to_slave() {
    echo "$(date): Switching to slave due to high CPU" | tee -a $LOG_FILE
    echo "disable server mysql-cluster/mysql-master" | docker exec -i mysql_loadbalancer socat stdio /tmp/haproxy.sock
    echo "enable server mysql-cluster/mysql-slave" | docker exec -i mysql_loadbalancer socat stdio /tmp/haproxy.sock
}

switch_to_master() {
    echo "$(date): Switching back to master" | tee -a $LOG_FILE
    echo "enable server mysql-cluster/mysql-master" | docker exec -i mysql_loadbalancer socat stdio /tmp/haproxy.sock
    echo "disable server mysql-cluster/mysql-slave" | docker exec -i mysql_loadbalancer socat stdio /tmp/haproxy.sock
}

# Main monitoring loop
current_state="master"
while true; do
    CPU_USAGE=$(get_cpu_usage)
    echo "$(date): CPU: ${CPU_USAGE}% | State: $current_state" | tee -a $LOG_FILE
    
    if [ "$CPU_USAGE" -gt "$CPU_THRESHOLD" ] && [ "$current_state" = "master" ]; then
        switch_to_slave
        current_state="slave"
    elif [ "$CPU_USAGE" -lt $((CPU_THRESHOLD - 20)) ] && [ "$current_state" = "slave" ]; then
        switch_to_master
        current_state="master"
    fi
    
    sleep $CHECK_INTERVAL
done
EOF

chmod +x cpu_monitor.sh
```

### Step 8: Testing Scripts

Create comprehensive testing scripts:

```bash
# Events page testing script
cat > test_events_pages.sh << 'EOF'
#!/bin/bash

BASE_URL="http://localhost:9007/events.php"
START_PAGE=2
END_PAGE=500
MAX_CONCURRENT=10
LOG_FILE="events_test_$(date +%Y%m%d_%H%M%S).log"

echo "=== Events Pages Test ===" | tee $LOG_FILE
echo "Testing pages $START_PAGE to $END_PAGE" | tee -a $LOG_FILE
echo "Started: $(date)" | tee -a $LOG_FILE

# Function to test a page
test_page() {
    local page=$1
    local url="${BASE_URL}?page=${page}"
    
    if curl -s --max-time 30 "$url" > /dev/null; then
        echo "Page $page: ✅" | tee -a $LOG_FILE
        return 0
    else
        echo "Page $page: ❌" | tee -a $LOG_FILE
        return 1
    fi
}

# Test pages with concurrent processing
success=0
failed=0
active_jobs=0

for page in $(seq $START_PAGE $END_PAGE); do
    while [ $active_jobs -ge $MAX_CONCURRENT ]; do
        wait -n
        active_jobs=$((active_jobs - 1))
    done
    
    (
        if test_page $page; then
            echo "SUCCESS" > /tmp/result_${page}
        else
            echo "FAILED" > /tmp/result_${page}
        fi
    ) &
    
    active_jobs=$((active_jobs + 1))
    
    if [ $((page % 50)) -eq 0 ]; then
        echo "Progress: $page pages started" | tee -a $LOG_FILE
    fi
done

wait

# Count results
for page in $(seq $START_PAGE $END_PAGE); do
    if [ -f /tmp/result_${page} ]; then
        if grep -q "SUCCESS" /tmp/result_${page}; then
            success=$((success + 1))
        else
            failed=$((failed + 1))
        fi
        rm /tmp/result_${page}
    fi
done

echo "=== Test Results ===" | tee -a $LOG_FILE
echo "Successful: $success" | tee -a $LOG_FILE
echo "Failed: $failed" | tee -a $LOG_FILE
echo "Success Rate: $(echo "scale=2; $success * 100 / ($success + $failed)" | bc)%" | tee -a $LOG_FILE
echo "Completed: $(date)" | tee -a $LOG_FILE
EOF

chmod +x test_events_pages.sh
```

## 🔧 Installation Process

### Complete Setup Commands

```bash
# 1. Clone and setup
git clone <your-repo> /www/docker/mysql-loadbalancer
cd /www/docker/mysql-loadbalancer

# 2. Update server IP in .env file
sed -i "s/YOUR_SERVER_IP/$(curl -s ifconfig.me)/" .env

# 3. Deploy services
./deploy.sh

# 4. Start CPU monitoring
nohup ./cpu_monitor.sh &

# 5. Verify installation
./verify_deployment.sh

# 6. Test events pages
./test_events_pages.sh
```

## 📊 Monitoring and Management

### Access Points

| Service | URL | Credentials |
|---------|-----|-------------|
| PHP App | `http://YOUR_IP:9007/` | None |
| HAProxy Stats | `http://YOUR_IP:8080/stats` | admin/admin123 |
| phpMyAdmin | `http://YOUR_IP:9008/` | root/rootpassword123 |
| phpMyAdmin Slave | `http://YOUR_IP:9009/` | root/rootpassword123 |

### Useful Commands

```bash
# Check container status
docker compose ps

# View logs
docker compose logs -f haproxy
docker compose logs -f mysql1
docker compose logs -f mysql2

# Monitor CPU and failover
tail -f /var/log/mysql_cpu_monitor.log

# Test MySQL connections
mysql -h localhost -P 3309 -uroot -prootpassword123 -e "SELECT @@server_id, @@hostname;"

# Manual failover
echo "disable server mysql-cluster/mysql-master" | docker exec -i mysql_loadbalancer socat stdio /tmp/haproxy.sock
echo "enable server mysql-cluster/mysql-slave" | docker exec -i mysql_loadbalancer socat stdio /tmp/haproxy.sock

# Restart services
docker compose restart
```

## 🔍 Troubleshooting

### Common Issues

1. **HAProxy not starting**
   ```bash
   # Check configuration
   docker run --rm -v $(pwd)/haproxy:/tmp/haproxy haproxy:2.8 haproxy -c -f /tmp/haproxy/haproxy.cfg
   
   # Check logs
   docker compose logs haproxy
   ```

2. **MySQL connection issues**
   ```bash
   # Test individual servers
   mysql -h localhost -P 3307 -uroot -prootpassword123 -e "SELECT 1"
   mysql -h localhost -P 3308 -uroot -prootpassword123 -e "SELECT 1"
   ```

3. **CPU monitoring not working**
   ```bash
   # Check if script is running
   ps aux | grep cpu_monitor
   
   # Restart monitoring
   pkill -f cpu_monitor
   nohup ./cpu_monitor.sh &
   ```

## 🔐 Security Considerations

1. **Change default passwords**
2. **Configure firewall rules**
3. **Use SSL/TLS certificates**
4. **Regular security updates**
5. **Monitor access logs**

## 📈 Performance Tuning

1. **Adjust HAProxy timeouts**
2. **Optimize MySQL configuration**
3. **Monitor resource usage**
4. **Scale horizontally when needed**

## 🆘 Support

For issues and support:
1. Check logs in `/var/log/` directory
2. Review container logs with `docker compose logs`
3. Monitor system resources with `htop`
4. Test network connectivity between containers

---

**Last Updated**: $(date)
**Version**: 1.0.0
