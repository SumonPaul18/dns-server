অবশ্যই! নিচে আপনার জন্য **PowerDNS ডকারে রান + অটোমেটেড কনফিগারেশন + .env ফাইল ব্যবহার + সম্ভাব্য সমস্যা ও সমাধান** — সবকিছু **Markdown ফরম্যাটে** একটি **সম্পূর্ণ README.md** ফাইল হিসেবে দিলাম।

এটি কপি করে সরাসরি `README.md` ফাইল হিসেবে সেভ করতে পারেন — বা গিটহাব/গিটল্যাব রিপোজিটরিতে ব্যবহার করতে পারেন।

---

# 📄 `README.md`

```markdown
# 🚀 PowerDNS in Docker — Simple, Automated, Production-Ready Setup

> ✅ এক কমান্ডে PowerDNS + Admin GUI রান  
> ✅ `.env` ফাইল দিয়ে সব কনফিগ ম্যানেজ  
> ✅ ওয়েব ইন্টারফেসে ক্লিক করে DNS রেকর্ড ম্যানেজ  
> ✅ সম্ভাব্য সমস্যা ও সমাধান সহ (পোর্ট 53 কনফ্লিক্ট ইত্যাদি)

---

## 🧩 প্রয়োজনীয় সফটওয়্যার

- Docker
- Docker Compose
- লিনাক্স/ম্যাক/WSL (Windows-এ চালানো যাবে)

---

## 📂 প্রজেক্ট স্ট্রাকচার

```
powerdns-simple/
├── .env                # সব কনফিগ ও সিক্রেট
├── docker-compose.yml  # ডকার কম্পোজ
└── README.md           # এই ফাইল
```

---

## 🔧 ধাপ ১: ফোল্ডার তৈরি ও ফাইল সেটআপ

```bash
mkdir powerdns-simple
cd powerdns-simple
```

---

## 📄 ধাপ ২: `.env` ফাইল তৈরি করুন

```bash
nano .env
```

নিচের কন্টেন্ট **কপি-পেস্ট** করুন:

```env
# MySQL ডাটাবেস সেটিংস
MYSQL_ROOT_PASSWORD=rootpass123
MYSQL_DATABASE=pdns
MYSQL_USER=pdns_user
MYSQL_PASSWORD=pdns_pass456

# PowerDNS API কী (GUI/API এক্সেসের জন্য)
PDNS_API_KEY=secretapikey

# পোর্ট ম্যাপিং
# ⚠️ পোর্ট 53 ব্যবহার করা যাচ্ছে না? → 5353 ব্যবহার করুন (নিচে সমাধান দেখুন)
PDNS_HOST_PORT=5353
PDNS_ADMIN_PORT=8080
PDNS_API_PORT=8081
```

> ✅ ভবিষ্যতে যেকোনো পরিবর্তন — শুধু `.env` ফাইল এডিট করুন → `docker-compose up -d` — অটো আপডেট!

---

## 🐳 ধাপ ৩: `docker-compose.yml` তৈরি করুন

```bash
nano docker-compose.yml
```

নিচের কন্টেন্ট **কপি-পেস্ট** করুন:

```yaml
version: '3.8'

services:
  # MySQL Database
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
      - mysql_data:/var/lib/mysql
    restart: unless-stopped

  # PowerDNS Server
  pdns:
    image: powerdns/pdns-auth-48
    container_name: pdns
    depends_on:
      - pdns-mysql
    env_file: .env
    environment:
      PDNS_launch: gmysql
      PDNS_gmysql_host: pdns-mysql
      PDNS_gmysql_user: ${MYSQL_USER}
      PDNS_gmysql_password: ${MYSQL_PASSWORD}
      PDNS_gmysql_dbname: ${MYSQL_DATABASE}
      PDNS_api: "yes"
      PDNS_api_key: ${PDNS_API_KEY}
      PDNS_webserver: "yes"
      PDNS_webserver_address: 0.0.0.0
      PDNS_webserver_port: ${PDNS_API_PORT}
      PDNS_webserver_allow_from: 0.0.0.0/0
    ports:
      - "${PDNS_HOST_PORT}:53/udp"
      - "${PDNS_HOST_PORT}:53/tcp"
      - "${PDNS_API_PORT}:8081/tcp"
    restart: unless-stopped

  # PowerDNS Admin Web GUI
  pdns-admin:
    image: ngoduykhanh/powerdns-admin:latest
    container_name: pdns-admin
    depends_on:
      - pdns
      - pdns-mysql
    env_file: .env
    environment:
      PDNS_ADMIN_SQLA_DB_HOST: pdns-mysql
      PDNS_ADMIN_SQLA_DB_PORT: 3306
      PDNS_ADMIN_SQLA_DB_USER: ${MYSQL_USER}
      PDNS_ADMIN_SQLA_DB_PASSWORD: ${MYSQL_PASSWORD}
      PDNS_ADMIN_SQLA_DB_NAME: ${MYSQL_DATABASE}
      PDNS_ADMIN_PDNS_STATS_URL: http://pdns:${PDNS_API_PORT}/
      PDNS_ADMIN_PDNS_API_KEY: ${PDNS_API_KEY}
      PDNS_ADMIN_PDNS_VERSION: 4.8.0
    ports:
      - "${PDNS_ADMIN_PORT}:80"
    restart: unless-stopped

volumes:
  mysql_data:
```

---

## 🚀 ধাপ ৪: ডকার কম্পোজ রান করুন

```bash
docker-compose up -d
```

> ⏳ প্রথমবার ২-৩ মিনিট সময় নিতে পারে — MySQL ইনিশিয়ালাইজ হচ্ছে।

---

## ✅ ধাপ ৫: চেক করুন — সব সার্ভিস চালু আছে কিনা?

```bash
docker-compose ps
```

আউটপুটে ৩টি সার্ভিস `Up` দেখালে — সব ঠিক আছে! ✅

---

## 🌐 ধাপ ৬: ওয়েব ইন্টারফেসে যান

ব্রাউজারে খুলুন:  
👉 **http://localhost:8080**

> 📌 প্রথমবার — রেজিস্ট্রেশন পেজ আসবে → একাউন্ট তৈরি করুন → লগইন করুন

---

## ➕ ধাপ ৭: প্রথম DNS জোন ও রেকর্ড যোগ করুন

1. **Dashboard → Domains → Add Domain**
   - Name: `mylab.local`
   - Type: `Native`
   - Create

2. **Records → Add Record**
   - Name: `www`
   - Type: `A`
   - Content: `192.168.1.100`
   - TTL: `3600`
   - Save

✅ রেকর্ড সেভ হয়ে গেছে — কোনো রিস্টার্ট লাগবে না!

---

## 🧪 ধাপ ৮: টেস্ট করুন

```bash
nslookup www.mylab.local 127.0.0.1 -port=5353
```

> ✅ যদি `.env` এ `PDNS_HOST_PORT=5353` সেট করে থাকেন — তাহলে `-port=5353` যোগ করুন।  
> ✅ যদি `PDNS_HOST_PORT=53` সেট করে থাকেন — তাহলে শুধু `nslookup www.mylab.local 127.0.0.1`

---

## ⚠️ সম্ভাব্য সমস্যা ও সমাধান

### ❌ সমস্যা 1: `failed to bind host port for 0.0.0.0:53: address already in use`

> 📌 **কারণ**: হোস্ট মেশিনে ইতিমধ্যে অন্য সার্ভিস (systemd-resolved, dnsmasq, bind) পোর্ট 53 ব্যবহার করছে।

#### ✅ সমাধান A: অন্য সার্ভিস বন্ধ করুন (সবচেয়ে সহজ)

```bash
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
sudo rm /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

তারপর:

```bash
docker-compose down
docker-compose up -d
```

#### ✅ সমাধান B: পোর্ট পরিবর্তন করুন (`.env` ফাইলে)

`.env` ফাইল এডিট করুন:

```env
PDNS_HOST_PORT=5353
```

তারপর:

```bash
docker-compose up -d
```

এবং টেস্ট করুন:

```bash
nslookup www.mylab.local 127.0.0.1 -port=5353
```

---

### ❌ সমস্যা 2: `pdns-admin` লোড হচ্ছে না

> 📌 **কারণ**: pdns বা mysql ঠিকমতো চালু হয়নি।

#### ✅ সমাধান:

```bash
docker-compose logs pdns
docker-compose logs pdns-mysql
```

→ লগ চেক করুন → সমস্যা ঠিক করুন → আবার `docker-compose up -d`

---

### ❌ সমস্যা 3: API কী ভুলে গেছেন?

→ `.env` ফাইলে `PDNS_API_KEY` চেক করুন → PowerDNS Admin এ লগইন → Settings → PDNS API Key আপডেট করুন।

---

## 🔄 ভবিষ্যতে কনফিগার করার নিয়ম

1. `.env` ফাইল এডিট করুন (পোর্ট, পাসওয়ার্ড, API key)
2. `docker-compose up -d` — অটো আপডেট হবে
3. রেকর্ড/জোন — শুধু ওয়েব ইন্টারফেসে গিয়ে এডিট করুন — অটো সেভ!

---

## 🧹 ক্লিনআপ (যদি রিসেট করতে চান)

```bash
docker-compose down -v
```

> ✅ `-v` দিলে ডাটাবেস ভলিউমও ডিলিট হবে — সবকিছু ফ্রেশ হবে!

---

## 🎁 বোনাস: কমান্ড শর্টকাট

| কাজ | কমান্ড |
|-----|--------|
| সার্ভিস চালু | `docker-compose up -d` |
| সার্ভিস বন্ধ | `docker-compose down` |
| লগ দেখুন | `docker-compose logs -f pdns` |
| স্ট্যাটাস চেক | `docker-compose ps` |
| রিস্টার্ট | `docker-compose restart pdns` |

---

## 📌 সারসংক্ষেপ

| বিষয় | বিবরণ |
|--------|--------|
| সেটআপ | ৩ ফাইল — `.env`, `docker-compose.yml`, `README.md` |
| কনফিগ | শুধু `.env` এডিট করুন |
| DNS ম্যানেজ | ওয়েব ইন্টারফেস (http://localhost:8080) |
| সমস্যা সমাধান | পোর্ট 53 → 5353, systemd-resolved বন্ধ |
| প্রডাকশন রেডি? | ✅ হ্যাঁ — সহজ, স্কেলেবল, মেইনটেইনেবল |

---

## 💬 পরবর্তী স্টেপ?

আপনি চাইলে:

1. **“PowerDNS কে রাউটারের DNS হিসেবে সেট করব কিভাবে?”**
2. **“PowerDNS এর জন্য SSL (HTTPS) কিভাবে যোগ করব?”**
3. **“PowerDNS API দিয়ে অটোমেশন করব কিভাবে?”**

> 📣 শুধু বলুন — “আমি চাই ________” — আমি আপনার জন্য পরবর্তী গাইড লিখে দেব।

---

🎉 **অভিনন্দন! আপনার PowerDNS সফলভাবে চালু হয়েছে!**  
Happy DNS Managing! 🌐
```

---

# ✅ কিভাবে ব্যবহার করবেন?

1. উপরের কন্টেন্ট কপি করুন
2. আপনার টার্মিনালে `nano README.md` করে পেস্ট করুন
3. সেভ করুন (`Ctrl+O` → `Enter` → `Ctrl+X`)
4. গিট রিপোতে কমিট করুন — বা শুধু রেফারেন্স হিসেবে রাখুন

---

# 🎯 এই সেটআপের সুবিধা

- **সহজ**: নতুনরাও বুঝবে
- **অটোমেটেড**: `.env` + `docker-compose`
- **ফ্লেক্সিবল**: পোর্ট কনফ্লিক্ট সমাধান ইন-বিল্ট
- **মেইনটেইনেবল**: কোনো ফাইল এডিট নয় — শুধু GUI বা `.env`
- **প্রডাকশন রেডি**: স্কেল করা যায়, ব্যাকআপ যোগ করা যায়, মনিটরিং যোগ করা যায়

---

ধন্যবাদ! 😊  
এই README.md আপনার পার্সোনাল বা টিম প্রজেক্টে ব্যবহার করতে পারেন — কোনো ক্রেডিট লাগবে না।  
আর কোনো প্রশ্ন থাকলে — জিজ্ঞাসা করুন, আমি আছি।
