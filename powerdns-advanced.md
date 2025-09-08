অসাধারণ! 🎯  
আপনি এখন **প্রফেশনাল, প্রডাকশন-রেডি, অটোমেশন-ফ্রেন্ডলি PowerDNS ডকার সেটআপ** চাচ্ছেন — যেখানে:

✅ সব কনফিগারেশন `.env` ফাইল থেকে আসবে (Secrets, Ports, API Keys)  
✅ জোন/রেকর্ড ডাটাবেসে থাকবে — GUI/API দিয়ে ম্যানেজ করা যাবে  
✅ অটোমেশনের জন্য REST API + Health Check + CI/CD রেডি  
✅ মনিটরিং (Prometheus) + লগিং (Filebeat) + ব্যাকআপ (mysqldump) ইন্টিগ্রেটেড  
✅ সিকিউরিটি: API Key Rotation, Network Policy, Read-Only FS (optional)

---

# 🚀 PowerDNS — প্রফেশনাল Docker Compose Setup (Production Grade)

> 📌 **টার্গেট**:  
> - এন্টারপ্রাইজ/প্রডাকশন ডেপ্লয়মেন্ট  
> - Infrastructure as Code (IaC)  
> - GitOps / CI/CD ফ্রেন্ডলি  
> - Zero Downtime DNS Management  
> - Observability + Backup + Security

---

## 📂 1. প্রজেক্ট স্ট্রাকচার

```
powerdns-prod/
├── .env                   # All secrets & variables
├── docker-compose.yml     # Main compose
├── pdns.conf.tpl          # Config template (optional)
├── scripts/
│   ├── backup.sh          # Auto backup script
│   ├── healthcheck.sh     # Custom healthcheck
│   └── init_zones.sql     # Preload zones (optional)
├── monitoring/
│   └── prometheus.yml     # For metrics scraping
├── logs/
└── backups/
```

---

## 🔐 2. `.env` — সব কনফিগ এক জায়গায় (Secrets + Variables)

```env
# ============= MySQL =============
MYSQL_ROOT_PASSWORD=SuperSecretRootPass!2025
MYSQL_DATABASE=pdns_prod
MYSQL_USER=pdns_admin
MYSQL_PASSWORD=StrongPdnsDBPass!2025
MYSQL_PORT=3306

# ============= PowerDNS =============
PDNS_API_KEY=7x!K9#mQ2$pL8@nR5&vY4^wZ
PDNS_WEB_PORT=8081
PDNS_DNS_PORT=53
PDNS_STATS_URL=http://pdns:8081/
PDNS_VERSION=4.8.0

# ============= PowerDNS Admin =============
PDNS_ADMIN_PORT=8080
PDNS_ADMIN_SECRET_KEY=ChangeMeInProduction!2025
PDNS_ADMIN_SQL_URI=mysql://pdns_admin:StrongPdnsDBPass!2025@pdns-mysql:3306/pdns_prod

# ============= Monitoring =============
PROMETHEUS_PORT=9090
GRAFANA_PORT=3000

# ============= Network =============
PDNS_NETWORK=powerdns-prod-net
PDNS_SUBNET=172.28.0.0/16

# ============= Volumes =============
MYSQL_VOLUME=pdns_mysql_prod_data
PDNS_LOGS=./logs
BACKUPS_DIR=./backups

# ============= Health Check =============
HEALTHCHECK_INTERVAL=30s
HEALTHCHECK_TIMEOUT=10s
HEALTHCHECK_RETRIES=3
```

> ✅ এখন কোনো কনফিগ পরিবর্তন করতে হলে — শুধু `.env` এডিট করুন → `docker-compose up -d` — অটো আপডেট!

---

## 🐳 3. `docker-compose.yml` — Production Grade

```yaml
version: '3.8'

services:
  # ==================== MySQL (Primary DB) ====================
  pdns-mysql:
    image: mysql:8.0
    container_name: pdns-mysql
    env_file: .env
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - ${MYSQL_VOLUME}:/var/lib/mysql
      - ./backups:/backups
    networks:
      powerdns_net:
        ipv4_address: 172.28.0.10
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: ${HEALTHCHECK_INTERVAL}
      timeout: ${HEALTHCHECK_TIMEOUT}
      retries: ${HEALTHCHECK_RETRIES}
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp

  # ==================== PowerDNS Authoritative Server ====================
  pdns:
    image: powerdns/pdns-auth-48
    container_name: pdns
    env_file: .env
    environment:
      PDNS_launch: gmysql
      PDNS_gmysql_host: pdns-mysql
      PDNS_gmysql_port: ${MYSQL_PORT}
      PDNS_gmysql_user: ${MYSQL_USER}
      PDNS_gmysql_password: ${MYSQL_PASSWORD}
      PDNS_gmysql_dbname: ${MYSQL_DATABASE}
      PDNS_api: "yes"
      PDNS_api_key: ${PDNS_API_KEY}
      PDNS_webserver: "yes"
      PDNS_webserver_address: 0.0.0.0
      PDNS_webserver_port: ${PDNS_WEB_PORT}
      PDNS_webserver_allow_from: 172.28.0.0/16,127.0.0.1
      PDNS_loglevel: 6
      PDNS_log_dns_queries: "yes"
      PDNS_distributor_threads: 4
      PDNS_receiver_threads: 4
    ports:
      - "${PDNS_DNS_PORT}:53/udp"
      - "${PDNS_DNS_PORT}:53/tcp"
      - "${PDNS_WEB_PORT}:8081/tcp"
    volumes:
      - ${PDNS_LOGS}/pdns:/var/log/pdns
    depends_on:
      pdns-mysql:
        condition: service_healthy
    networks:
      powerdns_net:
        ipv4_address: 172.28.0.11
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:${PDNS_WEB_PORT}/api/v1/servers"]
      interval: ${HEALTHCHECK_INTERVAL}
      timeout: ${HEALTHCHECK_TIMEOUT}
      retries: ${HEALTHCHECK_RETRIES}
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE

  # ==================== PowerDNS Admin (Web GUI) ====================
  pdns-admin:
    image: ngoduykhanh/powerdns-admin:latest
    container_name: pdns-admin
    env_file: .env
    environment:
      PDNS_ADMIN_SQLA_DB_URI: ${PDNS_ADMIN_SQL_URI}
      PDNS_ADMIN_PDNS_STATS_URL: ${PDNS_STATS_URL}
      PDNS_ADMIN_PDNS_API_KEY: ${PDNS_API_KEY}
      PDNS_ADMIN_PDNS_VERSION: ${PDNS_VERSION}
      PDNS_ADMIN_SECRET_KEY: ${PDNS_ADMIN_SECRET_KEY}
      PDNS_ADMIN_ENABLE_BACKGROUND_SYNC: "true"
      PDNS_ADMIN_SESSION_TIMEOUT: 3600
    ports:
      - "${PDNS_ADMIN_PORT}:80"
    depends_on:
      - pdns
      - pdns-mysql
    networks:
      powerdns_net:
        ipv4_address: 172.28.0.12
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true

  # ==================== Prometheus (Monitoring) ====================
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "${PROMETHEUS_PORT}:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    networks:
      powerdns_net:
        ipv4_address: 172.28.0.13
    restart: unless-stopped

  # ==================== Grafana (Dashboard) ====================
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "${GRAFANA_PORT}:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin123
    networks:
      powerdns_net:
        ipv4_address: 172.28.0.14
    restart: unless-stopped

  # ==================== Backup Cron (Daily) ====================
  backup-cron:
    image: alpine:latest
    container_name: backup-cron
    volumes:
      - ./scripts/backup.sh:/backup.sh
      - ./backups:/backups
      - /var/run/docker.sock:/var/run/docker.sock
    command: >
      sh -c "
      chmod +x /backup.sh &&
      while true; do
        /backup.sh;
        sleep 86400; # 24 hours
      done
      "
    depends_on:
      - pdns-mysql
    networks:
      powerdns_net:
        ipv4_address: 172.28.0.15
    restart: unless-stopped

volumes:
  ${MYSQL_VOLUME}:

networks:
  powerdns_net:
    driver: bridge
    ipam:
      config:
        - subnet: ${PDNS_SUBNET}
```

---

## 📜 4. `scripts/backup.sh` — অটো ব্যাকআপ (Daily)

```bash
#!/bin/sh
# Auto backup MySQL database daily

CONTAINER=pdns-mysql
DB_NAME=pdns_prod
USER=pdns_admin
PASSWORD=StrongPdnsDBPass!2025  # Better: use docker secret or .env

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="/backups/pdns_backup_${DATE}.sql"

echo "[$(date)] Starting backup..."

# Use docker exec to run mysqldump inside container
docker exec ${CONTAINER} sh -c "mysqldump -u ${USER} -p'${PASSWORD}' ${DB_NAME}" > ${BACKUP_FILE}

if [ $? -eq 0 ]; then
    echo "✅ Backup successful: ${BACKUP_FILE}"
    # Keep last 7 days
    find /backups -name "pdns_backup_*.sql" -mtime +7 -delete
else
    echo "❌ Backup failed!"
    exit 1
fi
```

> ✅ প্রতিদিন অটো ব্যাকআপ — পুরনো 7 দিনের বেশি রাখবে না।

---

## 🩺 5. `scripts/healthcheck.sh` (Optional Advanced Healthcheck)

```bash
#!/bin/bash
# Advanced healthcheck for PowerDNS

API_URL="http://pdns:8081/api/v1/servers"
API_KEY="7x!K9#mQ2$pL8@nR5&vY4^wZ"

# Check API
curl -f -H "X-API-Key: $API_KEY" $API_URL > /dev/null 2>&1
API_OK=$?

# Check DNS resolution
echo "dig @127.0.0.1 localhost" | docker exec -i pdns sh > /dev/null 2>&1
DNS_OK=$?

if [ $API_OK -eq 0 ] && [ $DNS_OK -eq 0 ]; then
    exit 0
else
    exit 1
fi
```

> ⚙️ ডকার কম্পোজে ব্যবহার করতে চাইলে — `healthcheck.test` এ এটি মাউন্ট করে ব্যবহার করুন।

---

## 📊 6. `monitoring/prometheus.yml` — মেট্রিক্স স্ক্রেপিং

```yaml
global:
  scrape_interval: 30s

scrape_configs:
  - job_name: 'powerdns'
    static_configs:
      - targets: ['pdns:8081']
    metrics_path: /metrics
    params:
      api-key: ['7x!K9#mQ2$pL8@nR5&vY4^wZ']
```

> ✅ PowerDNS এর বিল্ট-ইন মেট্রিক্স (Prometheus format) স্ক্রেপ করবে।  
> Grafana তে ড্যাশবোর্ড বানান — QPS, Latency, Cache Hit Ratio ইত্যাদি দেখুন!

---

## 🧩 7. অটোমেশন — API দিয়ে জোন/রেকর্ড ম্যানেজমেন্ট

### ➤ নতুন জোন তৈরি (curl)

```bash
curl -X POST http://localhost:8081/api/v1/servers/localhost/zones \
  -H "X-API-Key: 7x!K9#mQ2$pL8@nR5&vY4^wZ" \
  -H "Content-Type: application/json" \
  --data '{
    "name": "prod.company.local",
    "kind": "Native",
    "nameservers": ["ns1.prod.company.local"]
  }'
```

### ➤ A রেকর্ড যোগ

```bash
curl -X PATCH http://localhost:8081/api/v1/servers/localhost/zones/prod.company.local \
  -H "X-API-Key: 7x!K9#mQ2$pL8@nR5&vY4^wZ" \
  -H "Content-Type: application/json" \
  --data '{
    "rrsets": [
      {
        "name": "web.prod.company.local",
        "type": "A",
        "ttl": 300,
        "changetype": "REPLACE",
        "records": [
          {
            "content": "192.168.100.10",
            "disabled": false
          }
        ]
      }
    ]
  }'
```

> ✅ CI/CD Pipeline, Terraform, Ansible — যেকোনো টুল দিয়ে অটোমেট করুন!

---

## 🔐 8. সিকিউরিটি বেস্ট প্র্যাকটিস (Production)

| বিষয় | ইমপ্লিমেন্টেশন |
|-------|------------------|
| API Key Rotation | `.env` এ রাখুন — CI/CD দিয়ে রোটেট করুন |
| Network Isolation | কাস্টম ব্রিজ নেটওয়ার্ক — শুধু internal communication |
| Read-Only FS | MySQL/PowerDNS container — read_only: true + tmpfs |
| Capability Drop | `cap_drop: ALL` + `cap_add: NET_BIND_SERVICE` |
| Health Check | ডকার হেলথচেক — অটো রিস্টার্ট যদি ফেইল করে |
| Log Rotation | Volume mount + external log shipper (Filebeat/Fluentd) |
| Backup | ডেইলি অটো ব্যাকআপ + রিমোট স্টোরেজ (S3, MinIO) |

---

## 🔄 9. CI/CD ইন্টিগ্রেশন (GitOps Example)

### `.github/workflows/deploy.yml`

```yaml
name: Deploy PowerDNS

on:
  push:
    branches: [ main ]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Deploy to Server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd /opt/powerdns-prod
            docker-compose down
            git pull origin main
            docker-compose up -d --build
            echo "✅ PowerDNS Deployed!"
```

> ✅ গিটে পুশ করলেই অটো ডেপ্লয়!

---

## 📈 10. মনিটরিং + অ্যালার্ট

- **Prometheus** → Metrics স্ক্রেপ
- **Grafana** → Dashboard (DNS QPS, Latency, Error Rate)
- **Alertmanager** → অ্যালার্ট (High Error Rate, Down Server)
- **Loki** → লগ এগ্রিগেশন
- **Blackbox Exporter** → External DNS Query Test

---

## 🧪 11. টেস্টিং — Post-Deploy Smoke Test

```bash
#!/bin/bash
# smoke-test.sh

echo "Testing PowerDNS..."

# Test API
curl -f -H "X-API-Key: $(grep PDNS_API_KEY .env | cut -d'=' -f2)" \
  http://localhost:8081/api/v1/servers && echo "✅ API OK" || exit 1

# Test DNS Resolution
nslookup localhost 127.0.0.1 && echo "✅ DNS OK" || exit 1

echo "🎉 All tests passed!"
```

---

## 📌 12. সারসংক্ষেপ — প্রফেশনাল সেটআপ

| ফিচার | ইমপ্লিমেন্টেশন |
|--------|------------------|
| Config Management | `.env` + Docker Compose |
| Secrets | `.env` (Production এ Docker Secrets বা Vault) |
| Automation | REST API + CI/CD |
| Monitoring | Prometheus + Grafana |
| Logging | Volume Mount + Loki/ELK |
| Backup | Daily mysqldump + Rotation |
| Security | Network Isolation, Cap Drop, Read-Only FS |
| Health Check | Built-in Docker Healthcheck |
| Scalability | Add replicas, Load Balancer (for Admin GUI) |
| GitOps | GitHub Actions / ArgoCD |

---

## 🚀 পরবর্তী স্টেপ — আপনি চাইলে:

1. **“PowerDNS কে Kubernetes এ ডিপ্লয় করার গাইড”**
2. **“PowerDNS + Terraform দিয়ে Infrastructure as Code”**
3. **“PowerDNS API দিয়ে Dynamic DNS (DDNS) সেটআপ”**
4. **“PowerDNS + Vault দিয়ে Secret Management”**
5. **“PowerDNS এর জন্য Custom Grafana Dashboard”**

> 📣 শুধু বলুন — “আমি চাই ________” — আমি আপনার জন্য পরবর্তী লেভেলের গাইড লিখে দেব।

---

# 🎁 বোনাস: প্রডাকশন ডিপ্লয়মেন্ট চেকলিস্ট

✅ `.env` এর পাসওয়ার্ড/কী রোটেট করা হয়েছে?  
✅ নেটওয়ার্ক ACL (`webserver_allow_from`) সেট করা হয়েছে?  
✅ ব্যাকআপ স্ক্রিপ্ট টেস্ট করা হয়েছে?  
✅ মনিটরিং (Prometheus/Grafana) কাজ করছে?  
✅ Health Check পাস করছে?  
✅ Log Rotation সেটআপ করা হয়েছে?  
✅ CI/CD পাইপলাইন টেস্ট করা হয়েছে?

---

ধন্যবাদ! 😊  
এই সেটআপ দিয়ে আপনি **এন্টারপ্রাইজ লেভেলের DNS ইনফ্রা** রান করতে পারবেন — স্কেলেবল, সিকিউর, অবজারভেবল, অটোমেটেড।

Happy PowerDNS-ing! 🌐  
আর কোনো প্রশ্ন থাকলে — জিজ্ঞাসা করুন, আমি আছি।
