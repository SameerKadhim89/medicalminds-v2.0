# 🚀 دليل النشر الإنتاجي - MedicalMinds

## 📋 المتطلبات الأساسية

### الخادم (Server):
- ✅ Ubuntu 20.04+ / Debian 11+
- ✅ CPU: 2 cores minimum (4 cores recommended)
- ✅ RAM: 4GB minimum (8GB recommended)
- ✅ Storage: 50GB minimum (SSD recommended)
- ✅ Bandwidth: 1TB/month minimum

### البرامج المطلوبة:
```bash
# Docker & Docker Compose
sudo apt update
sudo apt install docker.io docker-compose

# Node.js 18+
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Nginx
sudo apt install nginx

# Certbot (SSL)
sudo apt install certbot python3-certbot-nginx

# Git
sudo apt install git
```

---

## 🔧 إعداد البيئة

### 1. Clone المشروع:
```bash
cd /var/www
git clone https://github.com/username/medicalminds.git
cd medicalminds
```

### 2. إعداد ملف Environment:
```bash
cd backend
cp .env.example .env
nano .env
```

### محتوى `.env`:
```env
# Application
NODE_ENV=production
PORT=3000
FRONTEND_URL=https://medicalminds.com

# Database
MONGODB_URI=mongodb://admin:password@mongodb:27017/medicalminds?authSource=admin
MONGO_ROOT_USER=admin
MONGO_ROOT_PASSWORD=your-secure-password
MONGO_DB_NAME=medicalminds

# Redis
REDIS_URL=redis://redis:6379

# JWT
JWT_SECRET=your-super-secret-jwt-key-min-32-chars
JWT_EXPIRES_IN=7d
JWT_REFRESH_SECRET=your-super-secret-refresh-key-min-32-chars
JWT_REFRESH_EXPIRES_IN=30d

# Email (SendGrid/SMTP)
EMAIL_HOST=smtp.sendgrid.net
EMAIL_PORT=587
EMAIL_USER=apikey
EMAIL_PASSWORD=your-sendgrid-api-key
EMAIL_FROM=noreply@medicalminds.com

# Upload
MAX_FILE_SIZE=10485760
UPLOAD_PATH=./uploads

# Security
RATE_LIMIT_WINDOW=15
RATE_LIMIT_MAX=100
BCRYPT_ROUNDS=12

# Monitoring
SENTRY_DSN=your-sentry-dsn (optional)
```

---

## 🐳 نشر مع Docker

### الطريقة 1: Development
```bash
# بناء وتشغيل
docker-compose up -d

# مشاهدة اللوقات
docker-compose logs -f

# إيقاف
docker-compose down
```

### الطريقة 2: Production
```bash
# بناء Production images
docker-compose -f docker-compose.yml -f docker-compose.prod.yml build

# تشغيل Production
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# التحقق من الحالة
docker-compose ps
```

---

## 🌐 إعداد Nginx

### 1. إنشاء ملف التكوين:
```bash
sudo nano /etc/nginx/sites-available/medicalminds
```

### محتوى الملف:
```nginx
# Upstream servers
upstream backend {
    server localhost:3000;
    keepalive 64;
}

# HTTP - Redirect to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name medicalminds.com www.medicalminds.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$server_name$request_uri;
    }
}

# HTTPS
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name medicalminds.com www.medicalminds.com;

    # SSL Configuration
    ssl_certificate /etc/letsencrypt/live/medicalminds.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/medicalminds.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Gzip Compression
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript application/json application/javascript application/xml+rss application/rss+xml font/truetype font/opentype application/vnd.ms-fontobject image/svg+xml;

    # Root directory
    root /var/www/medicalminds/frontend;
    index index.html;

    # API Proxy
    location /api/ {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # WebSocket Support
    location /socket.io/ {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # Static files
    location / {
        try_files $uri $uri/ /index.html;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Service Worker
    location /service-worker.js {
        add_header Cache-Control "no-cache";
        expires 0;
    }

    # Manifest
    location /manifest.json {
        add_header Cache-Control "no-cache";
        expires 0;
    }

    # Deny access to hidden files
    location ~ /\. {
        deny all;
    }
}
```

### 2. تفعيل الموقع:
```bash
sudo ln -s /etc/nginx/sites-available/medicalminds /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 🔒 إعداد SSL (Let's Encrypt)

### 1. الحصول على الشهادة:
```bash
sudo certbot --nginx -d medicalminds.com -d www.medicalminds.com
```

### 2. التجديد التلقائي:
```bash
# اختبار التجديد
sudo certbot renew --dry-run

# إضافة Cron Job
sudo crontab -e

# أضف السطر التالي:
0 0 * * * certbot renew --quiet && systemctl reload nginx
```

---

## 💾 إعداد قاعدة البيانات

### 1. تثبيت MongoDB:
```bash
# Import public key
wget -qO - https://www.mongodb.org/static/pgp/server-6.0.asc | sudo apt-key add -

# Create list file
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/6.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list

# Install
sudo apt update
sudo apt install -y mongodb-org

# Start
sudo systemctl start mongod
sudo systemctl enable mongod
```

### 2. إنشاء مستخدم قاعدة البيانات:
```bash
mongosh

use admin
db.createUser({
  user: "admin",
  pwd: "your-secure-password",
  roles: [ { role: "root", db: "admin" } ]
})

use medicalminds
db.createUser({
  user: "medicalminds",
  pwd: "your-app-password",
  roles: [ { role: "readWrite", db: "medicalminds" } ]
})

exit
```

### 3. Backup التلقائي:
```bash
# Create backup script
sudo nano /usr/local/bin/mongodb-backup.sh
```

```bash
#!/bin/bash
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/var/backups/mongodb"
mkdir -p $BACKUP_DIR

mongodump --uri="mongodb://admin:password@localhost:27017" --out="$BACKUP_DIR/backup_$TIMESTAMP"

# Keep only last 7 backups
find $BACKUP_DIR -type d -mtime +7 -exec rm -rf {} +
```

```bash
sudo chmod +x /usr/local/bin/mongodb-backup.sh

# Add to crontab
0 2 * * * /usr/local/bin/mongodb-backup.sh
```

---

## 📊 Monitoring & Logging

### 1. PM2 للعمليات:
```bash
npm install -g pm2

# Start application
pm2 start backend/server.js --name medicalminds

# Auto startup
pm2 startup
pm2 save

# Monitoring
pm2 monit
pm2 logs medicalminds
```

### 2. Prometheus & Grafana (Optional):
```yaml
# Add to docker-compose.yml
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3001:3000"
    volumes:
      - grafana_data:/var/lib/grafana
```

---

## 🔥 Firewall Setup

```bash
# UFW
sudo ufw allow 22/tcp   # SSH
sudo ufw allow 80/tcp   # HTTP
sudo ufw allow 443/tcp  # HTTPS
sudo ufw enable

# Fail2ban (Protection)
sudo apt install fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

---

## ✅ Checklist ما بعد النشر

### الأمان:
- [ ] تغيير جميع كلمات المرور الافتراضية
- [ ] تفعيل Firewall
- [ ] تفعيل SSL
- [ ] تكوين Fail2ban
- [ ] تفعيل تحديثات الأمان التلقائية

### الأداء:
- [ ] تفعيل Gzip
- [ ] تفعيل Browser Caching
- [ ] تحسين صور الموقع
- [ ] إعداد CDN (Cloudflare)

### Monitoring:
- [ ] إعداد Uptime monitoring
- [ ] إعداد Error tracking (Sentry)
- [ ] إعداد Analytics
- [ ] إعداد Backup تلقائي

### Testing:
- [ ] اختبار جميع المميزات
- [ ] اختبار SSL
- [ ] اختبار السرعة (Lighthouse)
- [ ] اختبار الأمان

---

## 🚨 Troubleshooting

### المشكلة: الموقع لا يعمل
```bash
# تحقق من حالة الخدمات
docker-compose ps
sudo systemctl status nginx
sudo systemctl status mongod

# تحقق من اللوقات
docker-compose logs
sudo journalctl -u nginx
```

### المشكلة: خطأ في قاعدة البيانات
```bash
# تحقق من الاتصال
mongosh --host localhost --port 27017

# إعادة تشغيل MongoDB
sudo systemctl restart mongod
```

### المشكلة: SSL لا يعمل
```bash
# تحقق من الشهادة
sudo certbot certificates

# تجديد الشهادة
sudo certbot renew
```

---

## 📞 الدعم

- 📧 Email: support@medicalminds.com
- 💬 Discord: [انضم لنا](https://discord.gg/medicalminds)
- 📖 Documentation: [docs.medicalminds.com](https://docs.medicalminds.com)

---

<div align="center">

**مبروك! 🎉 تطبيقك الآن في الإنتاج**

</div>
