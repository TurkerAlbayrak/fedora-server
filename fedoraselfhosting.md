# Fedora'da Website Self-Hosting Rehberi (Nginx ile)

Bu rehber, elinizdeki bir web sitesini (HTML/CSS/JS statik site veya PHP gibi basit bir uygulama) Fedora PC üzerinden nasıl yayınlayacağınızı, güncelleyeceğinizi ve yayından kaldıracağınızı anlatır. Tüm işlemler Fedora'nın konsolunda (terminal) yapılır; siteye Windows PC'den (aynı ağdan) tarayıcıyla erişeceksiniz. Önceki rehberdeki SSH/SCP bilgisiyle dosya güncellemesi de anlatılmıştır.

> Bu rehberde web sunucusu olarak **Nginx** kullanılmıştır (Apache'den daha hafif ve yaygın). İsterseniz Apache (`httpd`) de kullanılabilir, bölüm 8'de kısaca değinilmiştir.

---

## 1. Nginx Kurulumu

Fedora terminalinde:

```bash
sudo dnf install nginx -y
```

Kurulum bitince sürümü kontrol edin:

```bash
nginx -v
```

## 2. Nginx Servisini Başlatma

```bash
sudo systemctl start nginx
```

Fedora her açıldığında otomatik başlaması için:

```bash
sudo systemctl enable nginx
```

Servisin çalıştığını doğrulama:

```bash
sudo systemctl status nginx
```

`active (running)` yazısı görünmeli. Çıkmak için `q` tuşuna basabilirsiniz.

## 3. Firewall'da HTTP/HTTPS Portlarını Açma

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

Kontrol:

```bash
sudo firewall-cmd --list-services
```

Listede `http` ve `https` görünmeli.

## 4. SELinux Notu (Fedora'ya Özgü, Önemli!)

Fedora'da SELinux varsayılan olarak aktiftir ve Nginx'in dosyalara erişimini kısıtlayabilir. Site dosyalarınızı varsayılan dizin dışında bir yere koyarsanız (örn. `/home/kullanici/site`), SELinux izin vermeyebilir ve sitede **403 Forbidden** hatası alırsınız.

### Çözüm A — Dosyaları varsayılan dizine koymak (en kolay)

Nginx'in varsayılan site dizini: `/usr/share/nginx/html/`

### Çözüm B — Farklı bir dizin kullanmak istiyorsanız SELinux context ayarlama

```bash
sudo semanage fcontext -a -t httpd_sys_content_t "/home/kullanici/site(/.*)?"
sudo restorecon -Rv /home/kullanici/site
```

`semanage` komutu yoksa önce kurun:

```bash
sudo dnf install policycoreutils-python-utils -y
```

Eğer PHP gibi bir uygulama dosyaya yazma izni de istiyorsa (örn. upload klasörü):

```bash
sudo setsebool -P httpd_can_network_connect 1
sudo chcon -R -t httpd_sys_rw_content_t "/home/kullanici/site/uploads"
```

## 5. Website Dosyalarını Yerleştirme

### Basit yöntem: Varsayılan dizine kopyalama

Öncelikle eski varsayılan test sayfasını yedekleyin veya silin:

```bash
sudo mv /usr/share/nginx/html/index.html /usr/share/nginx/html/index.html.bak
```

Kendi sitenizin dosyalarını (örneğin `index.html`, `style.css`, `script.js`, resimler vb.) bu dizine kopyalayın:

```bash
sudo cp -r /home/kullanici/Downloads/mysite/* /usr/share/nginx/html/
```

Dosya sahipliğini ve izinlerini düzenleyin:

```bash
sudo chown -R nginx:nginx /usr/share/nginx/html/
sudo chmod -R 755 /usr/share/nginx/html/
```

### Alternatif: Özel bir dizin ve Nginx yapılandırması ile yayınlama (önerilen — daha temiz yönetim)

Kendi site klasörünüzü oluşturun:

```bash
sudo mkdir -p /var/www/mysite
sudo cp -r /home/kullanici/Downloads/mysite/* /var/www/mysite/
sudo chown -R nginx:nginx /var/www/mysite
sudo chmod -R 755 /var/www/mysite
```

SELinux context'i ayarlayın (Bölüm 4, Çözüm B'deki gibi):

```bash
sudo semanage fcontext -a -t httpd_sys_content_t "/var/www/mysite(/.*)?"
sudo restorecon -Rv /var/www/mysite
```

Yeni bir Nginx site konfigürasyonu oluşturun:

```bash
sudo nano /etc/nginx/conf.d/mysite.conf
```

İçine şunu yazın (nano içinde yapıştırma: sağ tık veya `Shift+Insert`; kaydetmek için `Ctrl+O`, Enter, çıkmak için `Ctrl+X`):

```nginx
server {
    listen 80;
    server_name _;

    root /var/www/mysite;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Yapılandırmanın hatasız olduğunu test edin:

```bash
sudo nginx -t
```

`syntax is ok` ve `test is successful` yazmalı. Ardından Nginx'i yeniden yükleyin:

```bash
sudo systemctl reload nginx
```

## 6. Windows PC'den Siteye Erişme

Önce Fedora'nın yerel ağ IP adresini öğrenin:

```bash
hostname -I
```

Örneğin `192.168.1.50` çıktıysa, Windows PC'de bir tarayıcı açıp adres çubuğuna:

```
http://192.168.1.50
```

yazın. Siteniz açılmalı.

> Not: Windows ve Fedora PC **aynı ağda (aynı Wi-Fi / switch)** olmalıdır.

## 7. Siteyi Güncelleme (Dosya Değiştirdiğinizde)

Windows'ta düzenlediğiniz dosyaları Fedora'ya SCP ile gönderin (önceki rehberdeki SSH bilgisiyle):

```powershell
scp -r C:\Users\Kullanici\Desktop\mysite-guncel\* kullaniciadi@192.168.1.50:/home/kullaniciadi/gelen-guncelleme/
```

Sonra Fedora tarafında (SSH ile bağlanıp) dosyaları asıl site klasörüne taşıyın:

```bash
sudo cp -r /home/kullaniciadi/gelen-guncelleme/* /var/www/mysite/
sudo chown -R nginx:nginx /var/www/mysite
sudo restorecon -Rv /var/www/mysite
```

Statik HTML/CSS/JS için Nginx'i yeniden başlatmanıza bile gerek yok — dosyalar anında yansır, tarayıcıda sayfayı yenilemeniz (Ctrl+F5 ile önbelleği atlayarak) yeterlidir.

Eğer `nginx.conf` veya `mysite.conf` gibi yapılandırma dosyalarında değişiklik yaptıysanız:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Git kullanıyorsanız (isteğe bağlı, daha profesyonel yöntem)

Eğer siteniz bir Git deposundaysa, güncellemeyi Fedora'da doğrudan çekebilirsiniz:

```bash
cd /var/www/mysite
sudo git pull origin main
sudo chown -R nginx:nginx /var/www/mysite
sudo restorecon -Rv /var/www/mysite
```

## 8. Siteyi Geçici Olarak Yayından Kaldırma (Durdurma)

### Yöntem 1: Nginx servisini tamamen durdurma (tüm siteler kapanır)

```bash
sudo systemctl stop nginx
```

Tekrar açmak için:

```bash
sudo systemctl start nginx
```

### Yöntem 2: Sadece bu siteyi devre dışı bırakma (Nginx açık kalır, diğer siteler etkilenmez)

Konfigürasyon dosyasını devre dışı bırakmanın en temiz yolu, uzantısını değiştirmektir (Nginx sadece `.conf` uzantılı dosyaları okur):

```bash
sudo mv /etc/nginx/conf.d/mysite.conf /etc/nginx/conf.d/mysite.conf.disabled
sudo nginx -t
sudo systemctl reload nginx
```

Tekrar aktif etmek için:

```bash
sudo mv /etc/nginx/conf.d/mysite.conf.disabled /etc/nginx/conf.d/mysite.conf
sudo nginx -t
sudo systemctl reload nginx
```

### Yöntem 3: "Bakımda" sayfası gösterme

`index.html` dosyasını geçici olarak bir bakım mesajıyla değiştirip orijinalini yedekleyebilirsiniz:

```bash
cd /var/www/mysite
sudo mv index.html index.html.orig
echo "<h1>Site bakımda, kısa süre sonra tekrar açılacak.</h1>" | sudo tee index.html
```

Geri almak için:

```bash
sudo mv index.html.orig index.html
```

## 9. Siteyi Tamamen Kaldırma (Silme)

```bash
sudo rm -rf /var/www/mysite
sudo rm /etc/nginx/conf.d/mysite.conf
sudo nginx -t
sudo systemctl reload nginx
```

Nginx'in kendisini de tamamen kaldırmak isterseniz:

```bash
sudo systemctl stop nginx
sudo systemctl disable nginx
sudo dnf remove nginx -y
```

## 10. Logları İnceleme (Sorun Giderme)

Erişim (kim, ne zaman, hangi sayfaya girdi) logları:

```bash
sudo tail -f /var/log/nginx/access.log
```

Hata logları (403, 404, 500 gibi sorunlarda buraya bakın):

```bash
sudo tail -f /var/log/nginx/error.log
```

Canlı izlemeyi durdurmak için `Ctrl + C`.

## 11. İnterneti Dış Dünyaya Açmak İstiyorsanız (Opsiyonel, Dikkat!)

Yukarıdaki adımlar sadece **yerel ağınızda** (ev/ofis Wi-Fi'ı) çalışır. Siteyi internete (dışarıya) açmak için ek adımlar ve güvenlik riskleri vardır:

1. **Router'da Port Forwarding**: Router yönetim paneline girip (genelde `192.168.1.1`), 80 (HTTP) ve 443 (HTTPS) portlarını Fedora PC'nizin yerel IP'sine yönlendirmeniz gerekir.
2. **Statik/Sabit Yerel IP**: Fedora'ya router üzerinden DHCP reservation ile sabit yerel IP atayın, aksi halde IP değiştiğinde port yönlendirmesi bozulur.
3. **Dinamik DNS (DDNS)**: Ev interneti genelde değişken (dinamik) genel IP kullanır; No-IP, DuckDNS gibi ücretsiz DDNS servisleriyle `siteniz.duckdns.org` gibi sabit bir adres alabilirsiniz.
4. **HTTPS/SSL Sertifikası**: Ücretsiz Let's Encrypt sertifikası için:
   ```bash
   sudo dnf install certbot python3-certbot-nginx -y
   sudo certbot --nginx -d alanadiniz.com
   ```
5. **Güvenlik Uyarısı**: Ev IP'nizi ve makinenizi internete açmak güvenlik riski taşır; güncellemeleri (`sudo dnf update`) düzenli yapın, gereksiz servisleri kapatın ve mümkünse `fail2ban` gibi bir araçla brute-force saldırılara karşı önlem alın:
   ```bash
   sudo dnf install fail2ban -y
   sudo systemctl enable --now fail2ban
   ```

> Eğer sadece ev/ofis ağınızda kendiniz veya aynı ağdaki cihazlar erişecekse (Bölüm 6'daki gibi), bu bölüm gerekli değildir.

## 12. Apache (httpd) ile Alternatif Kurulum (Kısa Özet)

Nginx yerine Apache tercih ederseniz:

```bash
sudo dnf install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

Varsayılan site dizini: `/var/www/html/`

```bash
sudo cp -r /home/kullanici/mysite/* /var/www/html/
sudo chown -R apache:apache /var/www/html/
sudo restorecon -Rv /var/www/html/
```

Durdurma:

```bash
sudo systemctl stop httpd
```

Yeniden başlatma / konfigürasyon testi:

```bash
sudo apachectl configtest
sudo systemctl restart httpd
```

---

## 13. Hızlı Komut Özeti

| İşlem | Komut |
|---|---|
| Nginx kur | `sudo dnf install nginx -y` |
| Başlat | `sudo systemctl start nginx` |
| Açılışta otomatik başlat | `sudo systemctl enable nginx` |
| Durdur (siteyi tamamen kapat) | `sudo systemctl stop nginx` |
| Yeniden başlat | `sudo systemctl restart nginx` |
| Config değişikliğini uygula (kesintisiz) | `sudo systemctl reload nginx` |
| Durum kontrolü | `sudo systemctl status nginx` |
| Config sözdizimi testi | `sudo nginx -t` |
| Firewall'da HTTP/HTTPS aç | `sudo firewall-cmd --permanent --add-service=http --add-service=https && sudo firewall-cmd --reload` |
| Fedora IP öğren | `hostname -I` |
| SELinux context düzelt | `sudo restorecon -Rv /var/www/mysite` |
| Erişim logu izle | `sudo tail -f /var/log/nginx/access.log` |
| Hata logu izle | `sudo tail -f /var/log/nginx/error.log` |
| Tek siteyi devre dışı bırak | `sudo mv mysite.conf mysite.conf.disabled` |
| Siteyi tamamen sil | `sudo rm -rf /var/www/mysite` |

---

**Not:** `192.168.1.50`, `mysite`, `kullaniciadi` gibi değerler örnektir; kendi ortamınızdaki gerçek değerlerle değiştirin.
