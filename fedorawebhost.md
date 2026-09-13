# Fedora'da React / Express (Node.js) Uygulamalarını Hostlama Rehberi

Önceki rehber statik HTML/CSS/JS siteler içindi. Bu rehber ise **Node.js tabanlı** (Express backend, React/Vue/Next.js frontend gibi) "büyük" veya dinamik siteleri Fedora'da nasıl çalıştırıp yayınlayacağınızı anlatır. Mantık şu şekilde çalışır:

- **React** (veya Vue, Angular vb.) → derlenip (`build`) statik dosyaya dönüşür → Nginx bunu doğrudan sunar.
- **Express** (veya Next.js, NestJS vb.) → bir Node.js süreci olarak arka planda **sürekli çalışır** (port 3000, 5000 gibi bir iç portta) → Nginx bu porta "reverse proxy" ile yönlendirme yapar.

Bu sayede dışarıya sadece 80/443 portu açık olur, Node uygulaması dış dünyaya doğrudan maruz kalmaz, Nginx önünde SSL/güvenlik katmanı sağlar.

---

## 1. Node.js Kurulumu

Fedora'da güncel Node.js sürümünü kurmak için NodeSource deposu (önerilen) veya Fedora'nın kendi paketi kullanılabilir.

### Yöntem A: Fedora'nın kendi deposu (basit ama sürüm eski olabilir)

```bash
sudo dnf install nodejs npm -y
```

### Yöntem B: NodeSource ile güncel sürüm (önerilen)

```bash
sudo dnf install curl -y
curl -fsSL https://rpm.nodesource.com/setup_lts.x | sudo bash -
sudo dnf install nodejs -y
```

Kurulumu doğrulama:

```bash
node -v
npm -v
```

---

## 2. Proje Dosyalarını Fedora'ya Aktarma

Windows'ta hazırladığınız projeyi (React/Express klasörünüzü) önceki rehberdeki SCP yöntemiyle gönderin:

```powershell
scp -r C:\Users\Kullanici\Desktop\mynodeapp kullaniciadi@192.168.1.50:/home/kullaniciadi/
```

Alternatif olarak, proje bir Git deposundaysa Fedora'da doğrudan klonlayabilirsiniz:

```bash
cd /home/kullaniciadi
git clone https://github.com/kullanici/mynodeapp.git
```

---

## 3. BÖLÜM A: React (veya Vue/Angular) — Statik Build Yayınlama

React gibi frontend framework'ler production'da **statik dosyaya derlenir**; yani sunucu tarafında sürekli çalışan bir Node süreci gerekmez, sadece Nginx yeterlidir.

### 3.1 Bağımlılıkları kurma ve build alma

Proje klasörüne girin:

```bash
cd /home/kullaniciadi/mynodeapp
npm install
npm run build
```

Bu işlem sonunda genelde bir `build/` (Create React App) veya `dist/` (Vite) klasörü oluşur. İçinde derlenmiş `index.html`, `.js`, `.css` dosyaları vardır.

### 3.2 Build dosyalarını Nginx'in servis edeceği dizine kopyalama

```bash
sudo mkdir -p /var/www/myreactapp
sudo cp -r /home/kullaniciadi/mynodeapp/build/* /var/www/myreactapp/
sudo chown -R nginx:nginx /var/www/myreactapp
sudo chmod -R 755 /var/www/myreactapp
```

SELinux context ayarı (önceki rehberdeki gibi):

```bash
sudo semanage fcontext -a -t httpd_sys_content_t "/var/www/myreactapp(/.*)?"
sudo restorecon -Rv /var/www/myreactapp
```

### 3.3 Nginx konfigürasyonu (React Router / SPA desteğiyle)

```bash
sudo nano /etc/nginx/conf.d/myreactapp.conf
```

İçeriği:

```nginx
server {
    listen 80;
    server_name _;

    root /var/www/myreactapp;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

> `try_files $uri $uri/ /index.html;` satırı önemlidir: React Router gibi client-side routing kullanan uygulamalarda, `/hakkimizda` gibi bir alt sayfaya doğrudan girildiğinde Nginx'in 404 vermemesi, her zaman `index.html`'e yönlendirmesi için gereklidir.

Test edip devreye alın:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### 3.4 React Projesini Güncelleme

Kod değiştiğinde tekrar build alıp kopyalamanız yeterli:

```bash
cd /home/kullaniciadi/mynodeapp
git pull            # veya Windows'tan yeni dosyaları SCP ile gönderin
npm install         # yeni paket eklendiyse
npm run build
sudo cp -r build/* /var/www/myreactapp/
sudo restorecon -Rv /var/www/myreactapp
```

Nginx'i yeniden başlatmaya gerek yok, dosyalar anında yansır (tarayıcıda `Ctrl+F5` ile önbelleği temizleyin).

---

## 4. BÖLÜM B: Express (veya Next.js/NestJS) — Sürekli Çalışan Node Süreci

Express gibi bir backend, tarayıcıdan her istek geldiğinde çalışan **sürekli aktif bir süreçtir**. Bu yüzden onu arka planda kalıcı şekilde çalıştırmak, çökerse otomatik yeniden başlatmak gerekir. Bunun için **PM2** (en yaygın yöntem) veya **systemd servisi** kullanılabilir.

### 4.1 Yöntem 1: PM2 ile Process Yönetimi (Kolay ve Popüler)

PM2'yi global olarak kurun:

```bash
sudo npm install -g pm2
```

Proje klasörüne girip bağımlılıkları kurun:

```bash
cd /home/kullaniciadi/mynodeapp
npm install
```

Uygulamayı PM2 ile başlatın (örnek giriş dosyası `server.js` veya `index.js` olabilir, kendi projenize göre değiştirin):

```bash
pm2 start server.js --name mynodeapp
```

Belirli bir port kullanıyorsa (örn. `.env` dosyasında `PORT=3000` tanımlıysa), Express uygulamanız zaten o portu dinleyecektir.

#### PM2 temel komutları

```bash
pm2 list                    # çalışan tüm uygulamaları listeler
pm2 status                  # durum özeti
pm2 logs mynodeapp          # canlı log izleme (Ctrl+C ile çık)
pm2 restart mynodeapp       # yeniden başlat
pm2 stop mynodeapp          # durdur (yayından kaldır)
pm2 start mynodeapp         # tekrar başlat
pm2 delete mynodeapp        # PM2 listesinden tamamen kaldır
```

#### Fedora yeniden başladığında PM2'nin otomatik açılması

```bash
pm2 startup systemd
```

Bu komut ekrana `sudo` ile çalıştırmanız gereken bir komut satırı yazdıracak (kullanıcı adınıza göre değişir), onu kopyalayıp çalıştırın. Ardından mevcut çalışan süreçleri kaydedin:

```bash
pm2 save
```

Artık Fedora yeniden başlatılsa bile `mynodeapp` otomatik ayağa kalkar.

### 4.2 Yöntem 2: systemd Servisi ile Çalıştırma (Daha "Linux-native" yöntem)

PM2 kurmak istemiyorsanız, Express uygulamanızı doğrudan bir systemd servisi olarak tanımlayabilirsiniz.

```bash
sudo nano /etc/systemd/system/mynodeapp.service
```

İçeriği:

```ini
[Unit]
Description=My Node/Express App
After=network.target

[Service]
Type=simple
User=kullaniciadi
WorkingDirectory=/home/kullaniciadi/mynodeapp
ExecStart=/usr/bin/node /home/kullaniciadi/mynodeapp/server.js
Restart=on-failure
Environment=NODE_ENV=production
Environment=PORT=3000

[Install]
WantedBy=multi-user.target
```

Servisi tanıtıp başlatın:

```bash
sudo systemctl daemon-reload
sudo systemctl start mynodeapp
sudo systemctl enable mynodeapp
```

Durum kontrolü:

```bash
sudo systemctl status mynodeapp
```

Logları görüntüleme:

```bash
sudo journalctl -u mynodeapp -f
```

Durdurma:

```bash
sudo systemctl stop mynodeapp
```

Yeniden başlatma (kod güncellemesi sonrası):

```bash
sudo systemctl restart mynodeapp
```

---

## 5. Nginx ile Express'e Reverse Proxy Yapılandırması

Node uygulamanız (PM2 veya systemd ile) örneğin `3000` portunda çalışıyor olsun. Dışarıdan (Windows tarayıcısından) `http://192.168.1.50` yazıldığında bu isteğin Node'a yönlendirilmesi için:

```bash
sudo nano /etc/nginx/conf.d/mynodeapp.conf
```

```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Test edip aktif edin:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Artık Windows'tan `http://192.168.1.50` adresine gidildiğinde Nginx, isteği arka planda 3000 portunda çalışan Express uygulamanıza iletecektir.

> `proxy_set_header Upgrade`/`Connection` satırları özellikle WebSocket kullanan uygulamalar (örn. Socket.io, canlı bildirim sistemleri) için gereklidir.

---

## 6. Tam Yığın (Full-Stack) Kurulum: React + Express Birlikte

Genelde React frontend'i Express API'sine istek atar. İki yaygın yaklaşım vardır:

### Yaklaşım 1: Ayrı yayınlama + Nginx'te path bazlı yönlendirme (önerilen)

React build'i statik olarak `/var/www/myreactapp` içinde, Express API'si ise `3000` portunda arka planda çalışır. Nginx'te:

```nginx
server {
    listen 80;
    server_name _;

    root /var/www/myreactapp;
    index index.html;

    # Frontend (React)
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Backend API (Express)
    location /api/ {
        proxy_pass http://127.0.0.1:3000/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Bu yapıda React'te API çağrılarınızı `/api/...` şeklinde yapmanız gerekir (örn. `fetch('/api/users')`), Express tarafında da route'larınızı buna göre tanımlarsınız.

### Yaklaşım 2: Express'in React build'ini kendisinin sunması (tek süreç)

Express uygulamanızda:

```javascript
const express = require('express');
const path = require('path');
const app = express();

app.use(express.static(path.join(__dirname, 'build')));

app.get('/api/ornek', (req, res) => {
  res.json({ mesaj: 'Merhaba API' });
});

app.get('*', (req, res) => {
  res.sendFile(path.join(__dirname, 'build', 'index.html'));
});

app.listen(3000, () => console.log('Sunucu 3000 portunda çalışıyor'));
```

Bu durumda Nginx sadece basit bir reverse proxy olarak (Bölüm 5'teki gibi) tüm trafiği 3000 portuna yönlendirir, ayrı bir React konfigürasyonuna gerek kalmaz.

---

## 7. Ortam Değişkenleri (.env) Yönetimi

Express projelerinde genelde veritabanı şifresi, API anahtarı gibi bilgiler `.env` dosyasında tutulur. Bu dosyayı Windows'tan SCP ile göndermeyi unutmayın (genelde `.gitignore`'da olduğu için Git'e dahil edilmez):

```powershell
scp .env kullaniciadi@192.168.1.50:/home/kullaniciadi/mynodeapp/
```

PM2 kullanıyorsanız `.env` dosyasını otomatik okuması için `dotenv` paketinin projede kurulu ve kodda `require('dotenv').config();` şeklinde çağrılmış olması gerekir.

systemd kullanıyorsanız, `.env` yerine servis dosyasına doğrudan `Environment=` satırları ekleyebilir veya `EnvironmentFile=/home/kullaniciadi/mynodeapp/.env` satırını `[Service]` bölümüne ekleyebilirsiniz.

---

## 8. Veritabanı Kurulumu (Opsiyonel — PostgreSQL / MySQL / MongoDB)

Büyük uygulamalarda genelde bir veritabanı da gerekir. Fedora'da kurulumu:

### PostgreSQL

```bash
sudo dnf install postgresql postgresql-server -y
sudo postgresql-setup --initdb
sudo systemctl enable --now postgresql
```

### MySQL / MariaDB

```bash
sudo dnf install mariadb mariadb-server -y
sudo systemctl enable --now mariadb
sudo mysql_secure_installation
```

### MongoDB

MongoDB, Fedora resmi depolarında bulunmayabilir; resmi MongoDB deposunu eklemeniz gerekebilir (MongoDB'nin kendi dokümantasyonundaki Fedora/RHEL kurulum adımları izlenmelidir).

> Veritabanı bağlantı bilgilerini (`.env` içinde) `localhost` (yani `127.0.0.1`) olarak ayarlayın, Node uygulaması ile aynı makinede çalıştığı için dışarıya açmanıza gerek yoktur — bu daha güvenlidir.

---

## 9. Güncelleme Akışı (Özet — Deploy Süreci)

Tipik bir güncelleme/deploy döngüsü:

```bash
cd /home/kullaniciadi/mynodeapp
git pull origin main
npm install                       # yeni bağımlılık varsa
npm run build                     # React/frontend build alınacaksa
sudo cp -r build/* /var/www/myreactapp/    # frontend güncelleniyorsa
sudo restorecon -Rv /var/www/myreactapp

# Backend'i yeniden başlat:
pm2 restart mynodeapp             # PM2 kullanıyorsanız
# veya
sudo systemctl restart mynodeapp  # systemd kullanıyorsanız
```

İsterseniz bu adımları tek bir `deploy.sh` scriptine yazıp SSH ile bağlanınca tek komutla çalıştırabilirsiniz:

```bash
nano ~/deploy.sh
```

```bash
#!/bin/bash
cd /home/kullaniciadi/mynodeapp
git pull origin main
npm install
npm run build
sudo cp -r build/* /var/www/myreactapp/
sudo restorecon -Rv /var/www/myreactapp
pm2 restart mynodeapp
echo "Deploy tamamlandı."
```

Çalıştırılabilir yapma ve çalıştırma:

```bash
chmod +x ~/deploy.sh
~/deploy.sh
```

---

## 10. Yayından Kaldırma / Durdurma

| Ne durdurulacak | Komut |
|---|---|
| Sadece Express backend (PM2) | `pm2 stop mynodeapp` |
| Sadece Express backend (systemd) | `sudo systemctl stop mynodeapp` |
| React statik siteyi devre dışı bırak | `sudo mv /etc/nginx/conf.d/myreactapp.conf /etc/nginx/conf.d/myreactapp.conf.disabled && sudo systemctl reload nginx` |
| Tüm web sunucusunu durdur (her şey kapanır) | `sudo systemctl stop nginx` |
| PM2'den tamamen sil | `pm2 delete mynodeapp` |
| systemd servisini tamamen kaldır | `sudo systemctl disable mynodeapp && sudo rm /etc/systemd/system/mynodeapp.service && sudo systemctl daemon-reload` |

---

## 11. Logları İzleme (Sorun Giderme)

```bash
pm2 logs mynodeapp                    # PM2 kullanıyorsanız
sudo journalctl -u mynodeapp -f       # systemd kullanıyorsanız
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

Node uygulaması hiç açılmıyorsa, portun gerçekten dinlenip dinlenmediğini kontrol edin:

```bash
sudo ss -tulpn | grep 3000
```

Bir satır dönerse (örn. `node` process'i 3000'i dinliyor) her şey yolunda demektir; hiçbir şey dönmezse uygulama düşmüş veya farklı bir portta çalışıyordur.

---

## 12. Kaynak Kullanımını İzleme (Büyük Uygulamalar İçin)

```bash
pm2 monit          # PM2 ile CPU/RAM canlı izleme
htop               # genel sistem kaynak izleme (yoksa: sudo dnf install htop -y)
```

---

## 13. Hızlı Komut Özeti

| İşlem | Komut |
|---|---|
| Node.js kur | `sudo dnf install nodejs npm -y` |
| Paketleri kur | `npm install` |
| React build al | `npm run build` |
| PM2 kur | `sudo npm install -g pm2` |
| PM2 ile başlat | `pm2 start server.js --name mynodeapp` |
| PM2 açılışta otomatik başlasın | `pm2 startup systemd` sonrası `pm2 save` |
| PM2 durdur | `pm2 stop mynodeapp` |
| PM2 yeniden başlat | `pm2 restart mynodeapp` |
| PM2 logları | `pm2 logs mynodeapp` |
| systemd ile başlat | `sudo systemctl start mynodeapp` |
| systemd yeniden başlat | `sudo systemctl restart mynodeapp` |
| systemd logları | `sudo journalctl -u mynodeapp -f` |
| Nginx reverse proxy testi | `sudo nginx -t && sudo systemctl reload nginx` |
| Port dinleniyor mu kontrol | `sudo ss -tulpn | grep 3000` |

---

**Not:** `mynodeapp`, `myreactapp`, `192.168.1.50`, `kullaniciadi`, `3000` gibi değerler örnektir; kendi projenize göre değiştirin. Bu rehber önceki "Fedora'da Website Self-Hosting Rehberi" ve "Windows-Fedora SSH Rehberi" dosyalarıyla birlikte kullanılmak üzere hazırlanmıştır.
