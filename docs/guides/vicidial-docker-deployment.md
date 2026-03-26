# Running VICIdial in Docker: The Container Setup Nobody Talks About

**You can absolutely run VICIdial in Docker. You probably shouldn't do it the way you're thinking. This guide covers the container architecture that actually works in production, the parts that fight you, and the configs that took us six months of broken calls to figure out.**

---

## Why This Article Exists

Every few months, someone on the VICIdial community forums asks "can I run VICIdial in Docker?" and gets one of two responses: an enthusiastic "yes, just containerize everything!" from someone who's never done it, or a flat "no, too many moving parts" from someone who tried once and gave up when Asterisk couldn't negotiate codecs through Docker's network stack.

Both answers are wrong. The truth is somewhere in the middle, and it's more nuanced than either camp admits.

We've been running containerized VICIdial components in production for two years. Not as a proof of concept. Not in a lab. In production, handling 400,000+ calls per month across three client operations. Some of those months were smooth. Some were 3 AM pages because a container restart killed 47 active calls.

This article is the guide we wish existed when we started. It covers what to containerize, what to leave on bare metal, how to handle the parts that don't want to be containerized, and the actual configuration files that make it work.

---

## The Architecture: What Goes in Containers and What Doesn't

Here's the first thing nobody tells you: you don't containerize all of VICIdial the same way. The system has very different components with very different requirements.

### Components That Containerize Well

1. **VICIdial Web Frontend (Apache/PHP)** — Stateless HTTP requests. Perfect container workload. Scale horizontally behind a load balancer. Zero issues.

2. **VICIdial Processes (Perl daemons)** — `AST_manager_send.pl`, `AST_manager_listen.pl`, `AST_VDauto_dial.pl`, `AST_VDadapt.pl`, and the other daemon scripts. They connect to MySQL and Asterisk AMI over TCP. They containerize fine as long as they can reach both endpoints.

3. **Ancillary Services** — Cron jobs, log rotation, the FastAGI server. All straightforward in containers.

### Components That Fight You

1. **Asterisk** — This is where it gets ugly. Asterisk needs raw UDP access for SIP signaling and RTP media. It needs a predictable, wide port range (10000-20000 by default). It does timing-sensitive operations that don't love container overhead. It can be containerized, but the network configuration is non-trivial.

2. **MySQL** — Databases in containers is a religious debate. VICIdial's MySQL handles high-write workloads (CDR inserts, real-time agent status updates). It works in Docker, but you absolutely need proper volume management and you should benchmark your I/O before going live.

3. **Recordings** — Call recordings generate gigabytes per day. They need persistent storage that survives container lifecycle. This is a volume management problem, not a Docker problem, but it catches people off guard.

### The Architecture We Actually Run

```
┌─────────────────────────────────────────────────┐
│                  Load Balancer                    │
│              (nginx / HAProxy)                    │
└──────────┬───────────────┬───────────────────────┘
           │               │
    ┌──────▼──────┐ ┌──────▼──────┐
    │  Web Front  │ │  Web Front  │    ← Docker containers
    │  (Apache)   │ │  (Apache)   │       (scale horizontally)
    └──────┬──────┘ └──────┬──────┘
           │               │
    ┌──────▼───────────────▼──────┐
    │        MySQL Primary         │    ← Docker or bare metal
    │      (persistent volume)     │       (your call)
    └──────┬───────────────┬──────┘
           │               │
    ┌──────▼──────┐ ┌──────▼──────┐
    │  Asterisk 1 │ │  Asterisk 2 │    ← Docker with host networking
    │  (dialer)   │ │  (dialer)   │       (NOT bridge mode)
    └─────────────┘ └─────────────┘
```

The critical insight: **Asterisk containers must use host networking mode.** Docker's default bridge networking adds NAT translation that breaks SIP in ways that are extremely difficult to debug. You'll get one-way audio, registration failures, and codec negotiation timeouts that make you question your career choices.

---

## Building the Docker Images

### Web Frontend Dockerfile

This is the straightforward one. VICIdial's web interface is PHP served by Apache. The only complexity is the Perl dependencies that some backend processes need.

```dockerfile
# VICIdial Web Frontend
FROM rockylinux:9

RUN dnf -y update && dnf -y install \
    httpd \
    php \
    php-mysqlnd \
    php-gd \
    php-mbstring \
    php-xml \
    perl \
    perl-DBI \
    perl-DBD-MySQL \
    perl-Digest-SHA \
    perl-Digest-HMAC \
    perl-MIME-Base64 \
    perl-Net-Telnet \
    perl-Time-HiRes \
    perl-Getopt-Long \
    perl-IO-Socket-SSL \
    perl-LWP-Protocol-https \
    perl-Net-SSLeay \
    && dnf clean all

# VICIdial source
COPY vicidial/ /srv/www/htdocs/vicidial/
COPY agc/ /srv/www/htdocs/agc/

# Apache config
COPY configs/httpd-vicidial.conf /etc/httpd/conf.d/vicidial.conf

# Database connection config
COPY configs/astguiclient.conf /etc/astguiclient.conf

# Health check endpoint
COPY configs/healthcheck.php /srv/www/htdocs/healthcheck.php

EXPOSE 80 443

CMD ["httpd", "-D", "FOREGROUND"]
```

The `/etc/astguiclient.conf` file is the single most important configuration file in the VICIdial ecosystem. Every Perl process reads it to find the database. In a containerized setup, you need to point it at your MySQL container's hostname:

```ini
# /etc/astguiclient.conf
VARDB_server => mysql
VARDB_database => asterisk
VARDB_user => cron
VARDB_pass => your_secure_password_here
VARDB_custom_user => custom
VARDB_custom_pass => your_custom_password_here
VARDB_port => 3306

VARserver_ip => CONTAINER_IP
VARactive_keepalives => 1234568
VARasterisk_version => 20
VARFTP_host => recordings
VARFTP_user => recordings
VARFTP_pass => recordings_password
VARHTTP_path => /vicidial
```

The `VARserver_ip` value is tricky in containers. On bare metal, it's the machine's IP. In Docker, it depends on your networking mode. With host networking, use the host IP. With bridge networking, use the container's bridge IP — but you'll need to handle service discovery.

### Asterisk Dockerfile

This is where the complexity lives.

```dockerfile
# VICIdial Asterisk Telephony
FROM rockylinux:9

RUN dnf -y update && dnf -y install \
    gcc gcc-c++ make \
    kernel-headers \
    ncurses-devel \
    libxml2-devel \
    sqlite-devel \
    openssl-devel \
    libuuid-devel \
    jansson-devel \
    libsrtp-devel \
    speex-devel \
    opus-devel \
    newt-devel \
    wget \
    perl \
    perl-DBI \
    perl-DBD-MySQL \
    perl-Digest-SHA \
    perl-Net-Telnet \
    perl-Time-HiRes \
    sox \
    lame \
    && dnf clean all

# Build Asterisk from source
ARG ASTERISK_VERSION=20.12.0
WORKDIR /usr/src
RUN wget -q https://downloads.asterisk.org/pub/telephony/asterisk/asterisk-${ASTERISK_VERSION}.tar.gz \
    && tar xzf asterisk-${ASTERISK_VERSION}.tar.gz \
    && cd asterisk-${ASTERISK_VERSION} \
    && contrib/scripts/get_mp3_source.sh \
    && ./configure --with-pjproject-bundled --with-jansson-bundled \
    && make menuselect.makeopts \
    && menuselect/menuselect \
        --enable CORE-SOUNDS-EN-ULAW \
        --enable MOH-OPSOUND-ULAW \
        --enable EXTRA-SOUNDS-EN-ULAW \
        --enable res_snmp \
        --enable chan_sip \
        menuselect.makeopts \
    && make -j$(nproc) \
    && make install \
    && make samples \
    && cd / && rm -rf /usr/src/asterisk-*

# VICIdial AGI scripts
COPY agi/ /var/lib/asterisk/agi-bin/
RUN chmod +x /var/lib/asterisk/agi-bin/*.agi

# Asterisk configs
COPY configs/asterisk/ /etc/asterisk/

# Recordings volume mount point
RUN mkdir -p /var/spool/asterisk/monitor

# VICIdial Perl daemons
COPY perl-scripts/ /usr/share/astguiclient/
COPY configs/astguiclient.conf /etc/astguiclient.conf

COPY entrypoint-asterisk.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

# SIP + RTP ports (only matters if not using host networking)
EXPOSE 5060/udp 5060/tcp 5061/tcp 10000-20000/udp

ENTRYPOINT ["/entrypoint.sh"]
```

The entrypoint script handles starting both Asterisk and the VICIdial Perl daemons:

```bash
#!/bin/bash
set -e

# Wait for MySQL to be available
echo "Waiting for MySQL..."
while ! perl -MDBI -e "DBI->connect('dbi:mysql:asterisk:mysql:3306','cron','${DB_PASS}')" 2>/dev/null; do
    sleep 2
done
echo "MySQL is up."

# Set server IP in astguiclient.conf
if [ -n "$SERVER_IP" ]; then
    sed -i "s/VARserver_ip =>.*/VARserver_ip => $SERVER_IP/" /etc/astguiclient.conf
fi

# Start Asterisk in background
asterisk -f -U asterisk -G asterisk &
ASTERISK_PID=$!

# Wait for Asterisk to be fully loaded
sleep 5
until asterisk -rx "core show version" >/dev/null 2>&1; do
    sleep 1
done
echo "Asterisk is running."

# Start VICIdial keepalive processes
cd /usr/share/astguiclient/
perl AST_manager_send.pl &
perl AST_manager_listen.pl &
perl AST_VDauto_dial.pl &
perl AST_VDadapt.pl &
perl AST_VDremote_agents.pl &
perl AST_update.pl &
perl AST_send_action_child.pl &

echo "VICIdial processes started."

# Keep container running, follow Asterisk
wait $ASTERISK_PID
```

### MySQL Dockerfile

For MySQL, you have two good options: use the official MySQL 8 image or build a custom one with VICIdial's schema pre-loaded.

```dockerfile
FROM mysql:8.0

# VICIdial requires specific SQL mode settings
COPY configs/my.cnf /etc/mysql/conf.d/vicidial.cnf

# Pre-load schema on first run
COPY sql/vicidial-schema.sql /docker-entrypoint-initdb.d/01-schema.sql
COPY sql/vicidial-initial-data.sql /docker-entrypoint-initdb.d/02-data.sql

# Tuning for call center workloads
# These get overridden by the .cnf file but are here for documentation
ENV MYSQL_ROOT_PASSWORD=changeme
ENV MYSQL_DATABASE=asterisk
```

The MySQL configuration matters a lot for VICIdial's write-heavy workload:

```ini
# configs/my.cnf
[mysqld]
innodb_buffer_pool_size = 4G
innodb_log_file_size = 512M
innodb_flush_log_at_trx_commit = 2
innodb_flush_method = O_DIRECT
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000

max_connections = 500
max_allowed_packet = 64M
tmp_table_size = 256M
max_heap_table_size = 256M

# VICIdial hits these tables hard
table_open_cache = 4000
thread_cache_size = 128

# Slow query logging — you will need this
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2

# Binary logging for replication (optional but recommended)
server-id = 1
log-bin = mysql-bin
binlog_expire_logs_seconds = 604800
```

---

## The Docker Compose Stack

Here's where it all comes together. This is the production `docker-compose.yml` we actually use:

```yaml
version: "3.8"

services:
  mysql:
    build:
      context: .
      dockerfile: Dockerfile.mysql
    container_name: vicidial-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: "${MYSQL_ROOT_PASS}"
      MYSQL_DATABASE: asterisk
      MYSQL_USER: cron
      MYSQL_PASSWORD: "${MYSQL_CRON_PASS}"
    volumes:
      - mysql_data:/var/lib/mysql
      - ./configs/my.cnf:/etc/mysql/conf.d/vicidial.cnf:ro
      - mysql_logs:/var/log/mysql
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - vicidial-internal

  web:
    build:
      context: .
      dockerfile: Dockerfile.web
    container_name: vicidial-web
    restart: unless-stopped
    depends_on:
      mysql:
        condition: service_healthy
    environment:
      DB_HOST: mysql
      DB_USER: cron
      DB_PASS: "${MYSQL_CRON_PASS}"
    volumes:
      - recordings:/var/spool/asterisk/monitor
      - web_logs:/var/log/httpd
    ports:
      - "443:443"
      - "80:80"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/healthcheck.php"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - vicidial-internal

  asterisk:
    build:
      context: .
      dockerfile: Dockerfile.asterisk
    container_name: vicidial-asterisk
    restart: unless-stopped
    depends_on:
      mysql:
        condition: service_healthy
    environment:
      SERVER_IP: "${HOST_IP}"
      DB_HOST: mysql
      DB_PASS: "${MYSQL_CRON_PASS}"
    volumes:
      - recordings:/var/spool/asterisk/monitor
      - asterisk_logs:/var/log/asterisk
    # THIS IS THE CRITICAL LINE
    network_mode: host
    # Since we're using host networking, we don't need port mappings
    # Asterisk binds directly to host ports 5060, 10000-20000

  recordings-nfs:
    image: erichough/nfs-server
    container_name: vicidial-nfs
    restart: unless-stopped
    volumes:
      - recordings:/data
    environment:
      NFS_EXPORT_0: "/data *(rw,sync,no_subtree_check,no_root_squash)"
    cap_add:
      - SYS_ADMIN
    ports:
      - "2049:2049"
    networks:
      - vicidial-internal

volumes:
  mysql_data:
    driver: local
  recordings:
    driver: local
    driver_opts:
      type: none
      device: /opt/vicidial/recordings
      o: bind
  mysql_logs:
    driver: local
  web_logs:
    driver: local
  asterisk_logs:
    driver: local

networks:
  vicidial-internal:
    driver: bridge
```

Notice the deliberate split: the web and MySQL containers use a bridge network for internal communication. The Asterisk container uses `network_mode: host` because it needs direct access to the host's network interfaces for SIP and RTP traffic.

This creates a problem: the Asterisk container can't use Docker DNS to resolve the `mysql` hostname. You need to configure the `astguiclient.conf` inside the Asterisk container to use the MySQL container's IP directly, or bind MySQL to the host network too, or use the Docker host's IP with the exposed MySQL port.

The cleanest solution:

```bash
# In entrypoint-asterisk.sh, resolve MySQL's container IP
MYSQL_IP=$(getent hosts mysql | awk '{print $1}')
# Falls back to the host IP if running in host network mode
if [ -z "$MYSQL_IP" ]; then
    MYSQL_IP="${HOST_IP}"
fi
sed -i "s/VARDB_server =>.*/VARDB_server => $MYSQL_IP/" /etc/astguiclient.conf
```

---

## Asterisk in a Container: The Hard Part

This section exists because we burned a lot of hours figuring this out. Asterisk in Docker works, but there are specific configuration requirements that differ from bare-metal installations.

### SIP Configuration (pjsip.conf)

When Asterisk runs inside a container — even with host networking — NAT detection can behave differently. Your `pjsip.conf` transport section needs explicit configuration:

```ini
; /etc/asterisk/pjsip.conf

[transport-udp]
type = transport
protocol = udp
bind = 0.0.0.0:5060
local_net = 10.0.0.0/8
local_net = 172.16.0.0/12
local_net = 192.168.0.0/16
external_media_address = YOUR_PUBLIC_IP
external_signaling_address = YOUR_PUBLIC_IP

[transport-tcp]
type = transport
protocol = tcp
bind = 0.0.0.0:5060
local_net = 10.0.0.0/8
local_net = 172.16.0.0/12
local_net = 192.168.0.0/16
external_media_address = YOUR_PUBLIC_IP
external_signaling_address = YOUR_PUBLIC_IP
```

The `local_net` entries are critical. Docker's bridge network uses `172.17.0.0/16` by default, which falls within the `172.16.0.0/12` range. Even with host networking, Asterisk may see Docker-related addresses and needs to know they're internal.

### RTP Port Range

VICIdial's default RTP range is 10000-20000. In a containerized environment, you should tighten this:

```ini
; /etc/asterisk/rtp.conf
[general]
rtpstart = 10000
rtpend = 15000
rtpchecksums = no
dtmftimeout = 3000
strictrtp = yes
```

Why shrink the range? Two reasons:
1. If you're NOT using host networking (against our advice, but sometimes necessary), you need to EXPOSE this entire range. Docker creates an iptables rule for every exposed port. 10,000 port mappings will slow container startup to a crawl and eat memory.
2. For most operations, 5,000 RTP ports supports 2,500 simultaneous calls. Unless you're running a massive operation, that's plenty.

### Timing Source

Asterisk needs a timing source for proper MusicOnHold, conferencing, and some dial plan operations. On bare metal, it uses `/dev/dahdi/timer` or the kernel's timerfd. In Docker:

```ini
; /etc/asterisk/modules.conf
[modules]
autoload = yes
noload => res_timing_dahdi.so    ; DAHDI isn't available in containers
load => res_timing_timerfd.so    ; Use timerfd instead — works everywhere
noload => chan_dahdi.so           ; No DAHDI hardware in containers
```

If you skip this, you'll get warnings about missing timing sources, and MusicOnHold will play at weird speeds or not at all. Agents on hold hearing chipmunk music is a real thing that happened to us.

### AMI Configuration

The VICIdial Perl daemons connect to Asterisk via AMI (Asterisk Manager Interface). In a containerized setup with host networking, this usually just works. But if you've got web containers on a bridge network that need AMI access:

```ini
; /etc/asterisk/manager.conf
[general]
enabled = yes
port = 5038
bindaddr = 0.0.0.0

[cron]
secret = your_ami_secret
read = all
write = all
writetimeout = 5000
deny = 0.0.0.0/0.0.0.0
; Allow Docker bridge network
permit = 172.16.0.0/255.240.0.0
; Allow localhost
permit = 127.0.0.1/255.255.255.255
; Allow host network
permit = 10.0.0.0/255.0.0.0
```

---

## Managing State and Recordings

### Recording Storage Architecture

Call recordings are the biggest storage consumer in any VICIdial deployment. A single agent generates roughly 2-4 GB of recordings per day depending on talk time and codec.

For a 50-agent operation running 8 hours, that's 100-200 GB of new recordings daily. You need a storage strategy that handles this volume and survives container restarts.

```yaml
# In docker-compose.yml — use a bind mount, not a Docker volume
volumes:
  recordings:
    driver: local
    driver_opts:
      type: none
      device: /mnt/recordings     # This is a separate disk / NFS mount
      o: bind
```

The bind mount approach is better than Docker named volumes for recordings because:
1. You can point it at a dedicated disk or NFS share
2. The files are accessible outside Docker for backup and archival
3. You can mount the same path in multiple containers (web for playback, Asterisk for writing)

For multi-server setups, NFS or object storage (MinIO, S3) is the right answer:

```bash
# Mount NFS on the Docker host, then bind-mount into containers
mount -t nfs recordings-server:/recordings /mnt/recordings

# Or use a Docker NFS volume driver
docker volume create --driver local \
    --opt type=nfs \
    --opt o=addr=192.168.1.100,rw,nfsvers=4 \
    --opt device=:/recordings \
    vicidial-recordings
```

### MySQL Data Persistence

For MySQL, use a named volume with a dedicated partition:

```bash
# Create a dedicated partition and format it
mkfs.xfs /dev/sdb1
mkdir -p /opt/mysql-data
mount /dev/sdb1 /opt/mysql-data

# Add to fstab
echo "/dev/sdb1 /opt/mysql-data xfs defaults,noatime 0 2" >> /etc/fstab
```

Then in your compose file:

```yaml
volumes:
  mysql_data:
    driver: local
    driver_opts:
      type: none
      device: /opt/mysql-data
      o: bind
```

### Backup Strategy

Containerized databases need backup too. Here's the cron job we run on the Docker host:

```bash
#!/bin/bash
# /etc/cron.d/vicidial-backup

# MySQL dump — runs inside the container
0 2 * * * docker exec vicidial-mysql mysqldump \
    -u root -p"${MYSQL_ROOT_PASS}" \
    --single-transaction \
    --routines \
    --triggers \
    asterisk | gzip > /opt/backups/vicidial-$(date +\%Y\%m\%d).sql.gz

# Keep 14 days of backups
0 3 * * * find /opt/backups/ -name "vicidial-*.sql.gz" -mtime +14 -delete

# Recording archival — move recordings older than 90 days to cold storage
0 4 * * 0 find /mnt/recordings/ -name "*.wav" -mtime +90 \
    -exec aws s3 mv {} s3://vicidial-recordings-archive/ \;
```

---

## Horizontal Scaling

One of the actual benefits of containerizing VICIdial is horizontal scaling. The web frontend scales trivially. Asterisk scales with some planning.

### Scaling the Web Tier

```bash
# Scale to 4 web frontend containers
docker compose up -d --scale web=4
```

Put an nginx or HAProxy in front:

```nginx
# /etc/nginx/conf.d/vicidial.conf
upstream vicidial_web {
    least_conn;
    server web-1:80;
    server web-2:80;
    server web-3:80;
    server web-4:80;
}

server {
    listen 443 ssl;
    server_name dialer.example.com;

    ssl_certificate /etc/ssl/certs/dialer.crt;
    ssl_certificate_key /etc/ssl/private/dialer.key;

    location / {
        proxy_pass http://vicidial_web;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # WebSocket support for real-time agent screens
    location /ws {
        proxy_pass http://vicidial_web;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### Scaling Asterisk

Asterisk scaling is different. You don't scale it the same way you scale web servers because each Asterisk instance needs:
- Its own SIP registration with your carrier
- Its own IP address (or port range) for RTP
- Its own entry in VICIdial's `servers` table in MySQL

This is exactly how VICIdial's multi-server architecture works on bare metal — it's a cluster pattern, not a scale-out pattern.

```bash
# Register each Asterisk server in VICIdial
# (Do this through the admin GUI at Admin > Servers, not direct SQL)

# Server 1: handles campaigns A and B
# Server 2: handles campaigns C and D
# Each server has its own carrier trunk config
```

The configuration in the VICIdial admin GUI under Admin > Servers lets you assign campaigns to specific Asterisk instances. Each container becomes a "server" in VICIdial's cluster model.

---

## Health Checks That Actually Work

Docker health checks for VICIdial need to check more than "is the process running." A container can have a running Apache process but broken PHP, or a running Asterisk process with no SIP registration.

### Web Container Health Check

```php
<?php
// /srv/www/htdocs/healthcheck.php
header('Content-Type: application/json');

$checks = [];
$healthy = true;

// Check MySQL connection
$db = @new mysqli(
    getenv('DB_HOST') ?: 'mysql',
    getenv('DB_USER') ?: 'cron',
    getenv('DB_PASS'),
    'asterisk'
);
if ($db->connect_error) {
    $checks['mysql'] = 'FAIL: ' . $db->connect_error;
    $healthy = false;
} else {
    // Actually run a query to verify the connection works
    $result = $db->query("SELECT COUNT(*) AS cnt FROM vicidial_campaigns");
    if ($result) {
        $row = $result->fetch_assoc();
        $checks['mysql'] = 'OK: ' . $row['cnt'] . ' campaigns';
    } else {
        $checks['mysql'] = 'FAIL: query error';
        $healthy = false;
    }
    $db->close();
}

// Check PHP extensions
$required_extensions = ['mysqlnd', 'gd', 'mbstring', 'xml'];
foreach ($required_extensions as $ext) {
    if (!extension_loaded($ext)) {
        $checks['php_' . $ext] = 'FAIL: not loaded';
        $healthy = false;
    }
}

http_response_code($healthy ? 200 : 503);
echo json_encode([
    'status' => $healthy ? 'healthy' : 'unhealthy',
    'checks' => $checks,
    'timestamp' => date('c')
]);
```

### Asterisk Container Health Check

```bash
#!/bin/bash
# /usr/local/bin/asterisk-healthcheck.sh

# Check 1: Is Asterisk responsive?
if ! asterisk -rx "core show version" >/dev/null 2>&1; then
    echo "FAIL: Asterisk not responding to CLI"
    exit 1
fi

# Check 2: Are SIP trunks registered?
REGISTERED=$(asterisk -rx "pjsip show registrations" 2>/dev/null | grep -c "Registered")
if [ "$REGISTERED" -eq 0 ]; then
    echo "FAIL: No SIP registrations active"
    exit 1
fi

# Check 3: Are VICIdial Perl daemons running?
for proc in AST_manager_send AST_manager_listen AST_VDauto_dial; do
    if ! pgrep -f "$proc" >/dev/null; then
        echo "FAIL: $proc not running"
        exit 1
    fi
done

# Check 4: Can we reach MySQL?
if ! perl -MDBI -e "DBI->connect('dbi:mysql:asterisk:${DB_HOST}:3306','cron','${DB_PASS}')" 2>/dev/null; then
    echo "FAIL: Cannot connect to MySQL"
    exit 1
fi

echo "OK: Asterisk + VICIdial daemons healthy"
exit 0
```

Add to the compose file:

```yaml
  asterisk:
    # ... other config ...
    healthcheck:
      test: ["CMD", "/usr/local/bin/asterisk-healthcheck.sh"]
      interval: 30s
      timeout: 15s
      retries: 3
      start_period: 60s   # Give Asterisk time to start
```

---

## Common Problems and How We Fixed Them

### Problem 1: One-Way Audio

**Symptom:** Agents can hear the lead but the lead can't hear the agent, or vice versa.

**Cause:** RTP packets are being sent to the wrong address because Asterisk doesn't know its external IP when running in a container.

**Fix:** Set `external_media_address` and `external_signaling_address` in `pjsip.conf` to your public IP. Also verify that your RTP port range is accessible through any firewalls.

```bash
# Verify RTP is flowing correctly
asterisk -rx "rtp set debug on"
# Look for "Sent RTP packet to" and verify the IP is correct
```

### Problem 2: Container Restart Kills Active Calls

**Symptom:** Docker restarting the Asterisk container drops every active call.

**Cause:** This is by design. There's no way to gracefully migrate active Asterisk channels between processes.

**Fix:** Don't let Docker restart your Asterisk container during business hours. Use a maintenance window approach:

```yaml
  asterisk:
    restart: unless-stopped
    deploy:
      update_config:
        order: start-first    # Start new container before stopping old
        failure_action: rollback
```

For zero-downtime updates, use VICIdial's multi-server architecture: drain calls from Server A, update Server A's container, shift calls back, then update Server B.

### Problem 3: MySQL Performance Degradation

**Symptom:** Agent screens become slow, real-time reports lag, calls take longer to connect.

**Cause:** Docker's storage driver (overlay2) adds overhead to disk I/O. Combined with VICIdial's write-heavy workload, you hit I/O limits faster than on bare metal.

**Fix:** Use direct bind mounts instead of Docker volumes. Put MySQL data on an SSD or NVMe. Tune `innodb_flush_log_at_trx_commit = 2` (it's in our config above) to batch commits instead of flushing every transaction.

```bash
# Check I/O performance inside the container
docker exec vicidial-mysql dd if=/dev/zero of=/tmp/iotest bs=1M count=1024 oflag=dsync
# Compare to the same test on the host
# If the container is >20% slower, your storage driver configuration needs work
```

### Problem 4: Perl Daemons Dying Silently

**Symptom:** Calls stop going out. The dialer looks stuck. Asterisk is running, MySQL is fine.

**Cause:** VICIdial's Perl daemons (`AST_VDauto_dial.pl` etc.) crash and don't restart because they're background processes inside the container.

**Fix:** Use a process supervisor inside the container. We use supervisord:

```ini
# /etc/supervisord.d/vicidial.ini
[program:asterisk]
command = /usr/sbin/asterisk -f -U asterisk -G asterisk
autorestart = true
startsecs = 10
stopwaitsecs = 30

[program:AST_manager_send]
command = /usr/bin/perl /usr/share/astguiclient/AST_manager_send.pl
autorestart = true
startsecs = 5

[program:AST_manager_listen]
command = /usr/bin/perl /usr/share/astguiclient/AST_manager_listen.pl
autorestart = true
startsecs = 5

[program:AST_VDauto_dial]
command = /usr/bin/perl /usr/share/astguiclient/AST_VDauto_dial.pl
autorestart = true
startsecs = 5

[program:AST_VDadapt]
command = /usr/bin/perl /usr/share/astguiclient/AST_VDadapt.pl
autorestart = true
startsecs = 5
```

Then change the Dockerfile's CMD:

```dockerfile
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]
```

---

## Production Deployment Checklist

Before you go live with containerized VICIdial, verify every item on this list. We learned each one the hard way.

- [ ] Asterisk container uses `network_mode: host` — no bridge networking for telephony
- [ ] `pjsip.conf` has correct `external_media_address` and `external_signaling_address`
- [ ] RTP port range is accessible through host firewall (`iptables -L -n | grep 10000`)
- [ ] MySQL data is on a bind-mounted SSD, not a Docker volume on the root disk
- [ ] Recording storage is on a separate mount that won't fill up the root filesystem
- [ ] Health checks verify actual functionality, not just process existence
- [ ] Perl daemons are managed by supervisord or equivalent process manager
- [ ] Container restart policy won't restart Asterisk during active calls without draining
- [ ] Backup cron job is running on the Docker host and verified with a test restore
- [ ] Log rotation is configured for Asterisk, Apache, and MySQL logs inside containers
- [ ] Time synchronization (NTP/chrony) is running on the Docker host — containers inherit host time
- [ ] `.env` file with passwords is NOT committed to version control

---

## Is It Worth It?

Honest answer: it depends on your operation.

**Containerize VICIdial if:**
- You're running multiple client operations and need isolation between them
- You have a DevOps team that understands Docker networking
- You want reproducible deployments across environments (dev/staging/prod)
- You're already in a container ecosystem (Kubernetes, ECS) and VICIdial needs to fit in

**Stay on bare metal if:**
- You're running a single operation with fewer than 100 agents
- Your team is strong on Linux administration but new to containers
- You need every last bit of telephony performance (bare metal Asterisk is still faster)
- You don't have a container orchestration platform already in place

For centers that want the performance of a tuned VICIdial installation without managing containers, infrastructure, or Asterisk configurations, [ViciStack](https://vicistack.com) handles the entire stack — containerized or bare metal — so you can focus on running campaigns instead of debugging Docker networking at 2 AM.

---

## Further Reading

- [VICIdial Cluster Setup Guide](https://vicistack.com/blog/vicidial-cluster-guide/) — multi-server architecture that applies to container scaling
- [VICIdial Performance Tuning](https://vicistack.com/blog/vicidial-performance-tuning/) — optimization techniques that apply whether you're on bare metal or containers
- [VICIdial Disaster Recovery](https://vicistack.com/blog/vicidial-disaster-recovery/) — backup and failover strategies for containerized deployments

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-docker-deployment).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
