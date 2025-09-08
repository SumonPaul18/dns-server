**PowerDNS** এবং **BIND9** কে **Docker Compose** দিয়ে রান করতে চান — এবং এমনভাবে যেন **যেকোনো DNS রেকর্ড (জোন, হোস্ট, IP) খুব সহজেই পরিবর্তন/যোগ/মুছে ফেলা যায়** — অর্থাৎ **Dynamic, Maintainable, Developer-Friendly Setup**।

নিচে **দুটি আলাদা ও সম্পূর্ণ স্টেপ-বাই-স্টেপ গাইড** দেওয়া আছে। 

---

# 🎯 গাইড 1: PowerDNS + Docker Compose (MySQL Backend + Admin GUI)

PowerDNS এর সবচেয়ে বড় সুবিধা — **API + Database Backend + Web GUI** — যার মানে:  
> ✅ জোন/রেকর্ড অটোমেটেডভাবে যোগ/পরিবর্তন করা যায় (REST API বা GUI দিয়ে)  
> ✅ কোনো ফাইল এডিট/রিস্টার্টের দরকার নেই!

---

## 🛠️ Step 1: প্রজেক্ট স্ট্রাকচার

```
pdns-docker/
├── docker-compose.yml
├── pdns.conf
├── .env
└── init.sql (optional)
```
```
mkdir pdns-docker && cd pdns-docker
```
---

## 📄 Step 2: `.env` ফাইল (কনফিগ ম্যানেজমেন্ট)

```
nano .env
```

```env
# MySQL Credentials
MYSQL_ROOT_PASSWORD=StrongRootPass123!
MYSQL_DATABASE=pdns
MYSQL_USER=pdns_user
MYSQL_PASSWORD=StrongPdnsPass456!

# PowerDNS API
PDNS_API_KEY=changeme-in-production
PDNS_WEB_PORT=8081
PDNS_DNS_PORT=53
```

> ✅ এখানে সব কনফিগ সেন্ট্রালাইজড — পরিবর্তন করতে হলে শুধু `.env` এডিট করুন।

---

## 📄 Step 3: `pdns.conf` — PowerDNS কনফিগ

```
nano pdns.conf
```

```ini
launch=gmysql
gmysql-host=pdns-mysql
gmysql-user=pdns_user
gmysql-password=StrongPdnsPass456!
gmysql-dbname=pdns

# API
api=yes
api-key=changeme-in-production
webserver=yes
webserver-address=0.0.0.0
webserver-port=8081
webserver-allow-from=0.0.0.0/0

# Logging
log-dns-queries=yes
loglevel=6

# DNS Settings
local-address=0.0.0.0
local-port=53
```

---

## 🐳 Step 4: `docker-compose.yml`

```
nano docker-compose.yml
```

```yaml
version: '3.8'

services:
  pdns-mysql:
    image: mysql:8.0
    container_name: pdns-mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - pdns_mysql_data:/var/lib/mysql
    networks:
      - pdns_net
    restart: unless-stopped

  pdns:
    image: powerdns/pdns-auth-48
    container_name: pdns
    depends_on:
      - pdns-mysql
    ports:
      - "${PDNS_DNS_PORT}:53/udp"
      - "${PDNS_DNS_PORT}:53/tcp"
      - "${PDNS_WEB_PORT}:8081/tcp"
    volumes:
      - ./pdns.conf:/etc/powerdns/pdns.conf
    environment:
      - PDNS_gmysql_host=pdns-mysql
      - PDNS_gmysql_user=${MYSQL_USER}
      - PDNS_gmysql_password=${MYSQL_PASSWORD}
      - PDNS_gmysql_dbname=${MYSQL_DATABASE}
      - PDNS_api=yes
      - PDNS_api_key=${PDNS_API_KEY}
      - PDNS_webserver=yes
      - PDNS_webserver_address=0.0.0.0
      - PDNS_webserver_port=8081
      - PDNS_webserver_allow_from=0.0.0.0/0
    networks:
      - pdns_net
    restart: unless-stopped

  pdns-admin:
    image: ngoduykhanh/powerdns-admin:latest
    container_name: pdns-admin
    depends_on:
      - pdns
    ports:
      - "80:80"
    environment:
      - PDNS_ADMIN_SQLA_DB_HOST=pdns-mysql
      - PDNS_ADMIN_SQLA_DB_PORT=3306
      - PDNS_ADMIN_SQLA_DB_USER=${MYSQL_USER}
      - PDNS_ADMIN_SQLA_DB_PASSWORD=${MYSQL_PASSWORD}
      - PDNS_ADMIN_SQLA_DB_NAME=pdns
      - PDNS_ADMIN_PDNS_STATS_URL=http://pdns:8081/
      - PDNS_ADMIN_PDNS_API_KEY=${PDNS_API_KEY}
      - PDNS_ADMIN_PDNS_VERSION=4.8.0
    networks:
      - pdns_net
    restart: unless-stopped

volumes:
  pdns_mysql_data:

networks:
  pdns_net:
    driver: bridge
```

---

## 🚀 Step 5: রান করুন

```bash
docker-compose up -d
```

---

## 🌐 Step 6: ওয়েব ইন্টারফেস

- **PowerDNS Admin GUI**: `http://localhost` (port 80)
- **PowerDNS API**: `http://localhost:8081` (API key দিয়ে অ্যাক্সেস)

> ✅ প্রথমবার লগইনে রেজিস্টার করুন — তারপর লগইন করে জোন/রেকর্ড ম্যানেজ করুন!

---

## ✅ কিভাবে নতুন জোন/রেকর্ড যোগ করবেন?

1. `http://localhost` — লগইন করুন
2. **Dashboard → Domains → Add Domain**
3. জোন নেম দিন (যেমন: `example.local`)
4. **Records → Add Record**
   - Name: `www`
   - Type: `A`
   - Content: `192.168.1.100`
   - TTL: `3600`

✅ সেভ করুন — **কোনো রিস্টার্ট/রিলোড লাগবে না!**  
DNS অটো আপডেট হয়ে যাবে।

---

## 🧪 টেস্ট করুন

```bash
nslookup www.example.local 127.0.0.1
dig @127.0.0.1 api.example.local
```

---

## 🔁 কিভাবে পরিবর্তন করবেন?

- `.env` — পোর্ট, পাসওয়ার্ড, API key পরিবর্তন
- `pdns.conf` — কনফিগ পরিবর্তন → তারপর `docker-compose restart pdns`
- **রেকর্ড/জোন** — শুধু ওয়েব ইন্টারফেসে গিয়ে এডিট করুন — অটো অ্যাপ্লাই!

---

# 🎯 গাইড 2: BIND9 + Docker Compose (Dynamic Zone with Git + Script)

BIND9 এর সবচেয়ে বড় সমস্যা — **জোন ফাইল ম্যানুয়ালি এডিট করে রিলোড করতে হয়**।

আমরা এটাকে **Dynamic** বানাবো — **Git + Script + Docker Volume + Auto-reload** দিয়ে!

---

## 🛠️ Step 1: প্রজেক্ট স্ট্রাকচার

```
bind-docker/
├── docker-compose.yml
├── named.conf
├── zones/
│   ├── db.example.local
│   └── db.192.168.1
├── reload.sh
└── .env
```

---

## 📄 Step 2: `.env`

```env
BIND_DNS_PORT=53
ZONE_DIR=./zones
CONFIG_DIR=./
```

---

## 📄 Step 3: `named.conf` (BIND মেইন কনফিগ)

```conf
options {
    directory "/var/cache/bind";
    listen-on port 53 { any; };
    allow-query { any; };
    recursion no;
    dnssec-validation no;
};

zone "example.local" {
    type master;
    file "/etc/bind/zones/db.example.local";
    allow-transfer { none; };
};

zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zones/db.192.168.1";
    allow-transfer { none; };
};
```

---

## 📄 Step 4: `zones/db.example.local`

```bind
$TTL 86400
@   IN  SOA ns1.example.local. admin.example.local. (
        2025040501  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        86400       ; Minimum TTL
)

@       IN  NS  ns1.example.local.
ns1     IN  A   192.168.1.10
www     IN  A   192.168.1.100
api     IN  A   192.168.1.101
```

---

## 📄 Step 5: `zones/db.192.168.1` (Reverse Zone)

```bind
$TTL 86400
@   IN  SOA ns1.example.local. admin.example.local. (
        2025040501  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        86400       ; Minimum TTL
)

@       IN  NS  ns1.example.local.
10      IN  PTR ns1.example.local.
100     IN  PTR www.example.local.
101     IN  PTR api.example.local.
```

---

## 📜 Step 6: `reload.sh` — অটো রিলোড স্ক্রিপ্ট

```bash
#!/bin/bash
# জোন ফাইল পরিবর্তন হলে BIND রিলোড করবে

# জোন ফাইলের সিরিয়াল অটো আপডেট
for zonefile in /etc/bind/zones/db.*; do
    if [ -f "$zonefile" ]; then
        # সিরিয়াল আপডেট (YYYYMMDDNN ফরম্যাট)
        DATE=$(date +%Y%m%d)
        LAST=$(grep -oP '\d{10}' "$zonefile" | tail -1)
        if [[ $LAST =~ ^${DATE} ]]; then
            # আজকের সিরিয়াল আছে — শেষ 2 ডিজিট +1
            NEW_SERIAL=$((LAST + 1))
        else
            # নতুন দিন — 00 দিয়ে শুরু
            NEW_SERIAL="${DATE}00"
        fi
        sed -i "s/\(.*SOA.*(\s*\)[0-9]\{10\}/\1$NEW_SERIAL/" "$zonefile"
    fi
done

# BIND রিলোড
echo "Reloading BIND zones..."
rndc reload
```

> ✅ এই স্ক্রিপ্ট জোন ফাইল এডিট করলে **অটো সিরিয়াল আপডেট + রিলোড** করবে!

---

## 🐳 Step 7: `docker-compose.yml`

```yaml
version: '3.8'

services:
  bind:
    image: internetsystemsconsortium/bind9:9.18
    container_name: bind-server
    ports:
      - "${BIND_DNS_PORT}:53/udp"
      - "${BIND_DNS_PORT}:53/tcp"
    volumes:
      - ${CONFIG_DIR}/named.conf:/etc/bind/named.conf
      - ${ZONE_DIR}:/etc/bind/zones
      - ./reload.sh:/reload.sh
    cap_add:
      - NET_ADMIN
    networks:
      - bind_net
    restart: unless-stopped
    command: >
      sh -c "
      chmod +x /reload.sh &&
      /usr/sbin/named -g -c /etc/bind/named.conf &
      while true; do
          inotifywait -e modify,create,delete /etc/bind/zones/ && /reload.sh;
      done
      "

volumes:
  bind_data:

networks:
  bind_net:
    driver: bridge
```

> ✅ `inotifywait` — জোন ফাইল পরিবর্তন হলে অটো `reload.sh` রান করবে!

---

## 🚀 Step 8: রান করুন

```bash
cd bind-docker
chmod +x reload.sh
docker-compose up -d
```

---

## ✅ কিভাবে নতুন রেকর্ড/জোন যোগ করবেন?

### ➤ নতুন রেকর্ড:

1. `zones/db.example.local` ফাইল এডিট করুন:

```bind
newhost    IN  A   192.168.1.200
```

2. **ফাইল সেভ করুন** → অটো রিলোড হবে!

### ➤ নতুন জোন:

1. `zones/db.newzone.local` ফাইল তৈরি করুন
2. `named.conf` এ নতুন জোন যোগ করুন:

```conf
zone "newzone.local" {
    type master;
    file "/etc/bind/zones/db.newzone.local";
};
```

3. সেভ করুন → অটো রিলোড!

---

## 🧪 টেস্ট করুন

```bash
nslookup newhost.example.local 127.0.0.1
dig @127.0.0.1 newzone.local
```

---

## 🔁 কিভাবে পরিবর্তন করবেন?

- জোন/রেকর্ড → শুধু `zones/` ফোল্ডারে ফাইল এডিট করুন → অটো রিলোড!
- পোর্ট/কনফিগ → `.env` বা `named.conf` এডিট → `docker-compose restart bind`

---

# 📊 PowerDNS vs BIND9 — ডায়নামিক ম্যানেজমেন্টের জন্য

| ফিচার | PowerDNS | BIND9 (আমাদের সেটআপ) |
|--------|----------|------------------------|
| ওয়েব GUI | ✅ হ্যাঁ (PowerDNS Admin) | ❌ না |
| API | ✅ REST API | ❌ না (rndc/script) |
| অটো রিলোড | ✅ (API/GUI) | ✅ (inotify + script) |
| ডাটাবেস | ✅ MySQL/PostgreSQL | ❌ ফাইল-ভিত্তিক |
| রেকর্ড যোগ | GUI/API — 1 ক্লিক | ফাইল এডিট — 2-3 লাইন |
| স্কেল | ✅ হাই | ⚠️ মিডিয়াম |
| লার্নিং কার্ভ | মিডিয়াম | হাই |

---

# ✅ উভয় সেটআপের সুবিধা

- **PowerDNS**: মডার্ন, API, GUI, অটোমেশন ফ্রেন্ডলি — প্রডাকশনের জন্য উত্তম।
- **BIND9**: লিগ্যাসি, ফাইল-ভিত্তিক — কিন্তু আমাদের সেটআপে **অটো রিলোড + সিরিয়াল ম্যানেজমেন্ট** যোগ করে দিয়েছি — তাই এখন এটিও ডায়নামিক!

---

# 📌 সারসংক্ষেপ

> ✅ **PowerDNS** — যদি আপনি চান:  
> - Web GUI  
> - REST API  
> - Database Backend  
> - Zero Downtime Update  

> ✅ **BIND9** — যদি আপনি চান:  
> - File-based Control  
> - Legacy System Support  
> - GitOps Friendly (জোন ফাইল Git এ রাখুন → CI/CD → Auto Deploy)  

> 🎯 **আপনার জন্য সবচেয়ে ভালো?**  
> - **প্রডাকশন/এন্টারপ্রাইজ → PowerDNS**  
> - **হোম ল্যাব/লার্নিং/GitOps → BIND9 (আমাদের সেটআপ)**

---

# 🚀 বোনাস: PowerDNS API দিয়ে অটোমেশন (curl example)

```bash
# নতুন জোন তৈরি
curl -X POST http://localhost:8081/api/v1/servers/localhost/zones \
  -H "X-API-Key: changeme-in-production" \
  -H "Content-Type: application/json" \
  --data '{"name":"example2.local", "kind": "Native", "nameservers": ["ns1.example2.local"]}'
```

```bash
# নতুন A রেকর্ড
curl -X PATCH http://localhost:8081/api/v1/servers/localhost/zones/example2.local \
  -H "X-API-Key: changeme-in-production" \
  -H "Content-Type: application/json" \
  --data '{"rrsets": [{"name": "app.example2.local", "type": "A", "ttl": 3600, "changetype": "REPLACE", "records": [{"content": "192.168.1.200", "disabled": false}]}]}'
```

---

# 💬 পরবর্তী স্টেপ?

আপনি চাইলে:

1. **PowerDNS Admin GUI এর স্ক্রিনশট + স্টেপ বাই স্টেপ গাইড**
2. **BIND9 এর জন্য CI/CD Pipeline (GitHub Actions + Auto Deploy)**
3. **PowerDNS কে Kubernetes এ ডিপ্লয় করার গাইড**
4. **DoH/DoT সেটআপ যোগ করা**

> 📣 শুধু বলুন — “আমি চাই ________” — আমি আপনার জন্য কাস্টমাইজড গাইড লিখে দেব।

---

ধন্যবাদ! 😊  
Happy DNS Managing! 🌐  
আর কোনো প্রশ্ন থাকলে — জিজ্ঞাসা করুন!
