# Windows'tan SSH ile Fedora ve Termux (Android) Bağlantı Rehberi

Bu rehber iki ana bölümden oluşur:

1. **Windows PC → Fedora PC** arası SSH bağlantısı (kurulum, bağlanma, çıkma, dosya transferi)
2. **Windows PC → Termux (Android)** arası SSH bağlantısı (OpenSSH kurulumu, bağlanma, dosya transferi)

---

# BÖLÜM 1: Windows'tan Fedora'ya SSH Bağlantısı

## 1.1 Fedora Tarafında Hazırlık (SSH Sunucusu Kurulumu)

Fedora'da genellikle OpenSSH sunucusu (sshd) ön yüklü gelir ama emin olmak için terminalde:

```bash
sudo dnf install openssh-server -y
```

### Servisi başlatma ve otomatik açılışta çalıştırma

```bash
sudo systemctl start sshd
sudo systemctl enable sshd
```

### Servis durumunu kontrol etme

```bash
sudo systemctl status sshd
```

`active (running)` yazısını görmelisiniz.

### Güvenlik duvarında (firewalld) SSH portunu açma

```bash
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

Kontrol etmek için:

```bash
sudo firewall-cmd --list-services
```

Listede `ssh` görünmeli.

### Fedora'nın IP adresini öğrenme

```bash
ip a
```

veya sadece yerel ağ IP'sini görmek için:

```bash
hostname -I
```

Çıktıda genelde `192.168.x.x` şeklinde bir adres göreceksiniz. Bu adres Windows'tan bağlanırken kullanılacak.

### Fedora'da kullanıcı adını öğrenme

```bash
whoami
```

## 1.2 Windows Tarafında Hazırlık

Windows 10 (1809 sonrası) ve Windows 11'de OpenSSH istemcisi genelde yüklüdür. Kontrol etmek için PowerShell veya CMD açın:

```powershell
ssh -V
```

Sürüm bilgisi çıkarsa yüklüdür. Çıkmazsa:

### PowerShell (Yönetici olarak) ile OpenSSH Client kurulumu

```powershell
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH*'
```

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
```

Kurulumdan sonra doğrulama:

```powershell
ssh -V
```

## 1.3 Fedora'ya SSH ile Bağlanma

Windows'ta PowerShell, CMD veya Windows Terminal açın:

```powershell
ssh kullaniciadi@192.168.1.50
```

- `kullaniciadi` → Fedora'daki kullanıcı adınız
- `192.168.1.50` → Fedora makinenin IP adresi

İlk bağlantıda şu şekilde bir uyarı çıkar (host key doğrulama):

```
The authenticity of host '192.168.1.50' can't be established.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

`yes` yazıp Enter'a basın. Ardından Fedora kullanıcınızın şifresini girin (şifre yazarken ekranda görünmez, bu normaldir).

### Farklı bir port kullanılıyorsa

Eğer Fedora'da SSH portu değiştirilmişse (varsayılan 22 değilse):

```powershell
ssh -p 2222 kullaniciadi@192.168.1.50
```

## 1.4 Bağlantıdan Çıkma

SSH oturumunu sonlandırmak için Fedora terminalinde (yani bağlandıktan sonraki ekranda) şunlardan birini yazın:

```bash
exit
```

veya

```bash
logout
```

veya klavye kısayolu olarak:

```
Ctrl + D
```

Bağlantı donmuşsa (yanıt vermiyorsa) SSH'ı zorla sonlandırmak için:

```
Enter tuşuna basın, sonra ~ ve ardından . tuşuna basın (Enter, ~, .)
```

## 1.5 SCP ile Dosya Gönderme / Alma

SCP (Secure Copy), SSH üzerinden dosya kopyalamak için kullanılır. Komutlar Windows tarafında (PowerShell/CMD) çalıştırılır.

### Windows → Fedora'ya tek dosya gönderme

```powershell
scp C:\Users\Kullanici\Desktop\dosya.txt kullaniciadi@192.168.1.50:/home/kullaniciadi/
```

### Fedora → Windows'a tek dosya alma (indirme)

```powershell
scp kullaniciadi@192.168.1.50:/home/kullaniciadi/dosya.txt C:\Users\Kullanici\Desktop\
```

### Klasör (dizin) gönderme — `-r` (recursive) parametresiyle

```powershell
scp -r C:\Users\Kullanici\Desktop\proje kullaniciadi@192.168.1.50:/home/kullaniciadi/
```

### Klasör alma

```powershell
scp -r kullaniciadi@192.168.1.50:/home/kullaniciadi/proje C:\Users\Kullanici\Desktop\
```

### Farklı port ile SCP kullanımı

```powershell
scp -P 2222 dosya.txt kullaniciadi@192.168.1.50:/home/kullaniciadi/
```

> Not: `ssh` komutunda küçük `-p`, `scp` komutunda büyük `-P` kullanılır.

### Birden fazla dosya gönderme

```powershell
scp dosya1.txt dosya2.txt kullaniciadi@192.168.1.50:/home/kullaniciadi/
```

## 1.6 SFTP ile Dosya Yönetimi (Alternatif Yöntem)

SFTP, interaktif bir dosya transfer oturumu açar (FTP benzeri ama şifreli).

Bağlanma:

```powershell
sftp kullaniciadi@192.168.1.50
```

SFTP oturumu içinde kullanılabilecek komutlar:

| Komut | Açıklama |
|---|---|
| `ls` | Uzak (Fedora) dizindeki dosyaları listeler |
| `lls` | Yerel (Windows) dizindeki dosyaları listeler |
| `cd klasor` | Uzak dizin değiştirir |
| `lcd klasor` | Yerel dizin değiştirir |
| `pwd` | Uzak mevcut dizini gösterir |
| `lpwd` | Yerel mevcut dizini gösterir |
| `get dosya.txt` | Fedora'dan dosya indirir |
| `put dosya.txt` | Windows'tan dosya yükler |
| `mkdir klasor` | Uzakta klasör oluşturur |
| `rm dosya.txt` | Uzakta dosya siler |
| `exit` veya `bye` | SFTP oturumunu kapatır |

## 1.7 SSH Anahtarı (Key) ile Şifresiz Giriş Ayarlama

Her seferinde şifre girmemek için SSH anahtar çifti oluşturup Fedora'ya kopyalayabilirsiniz.

### 1. Windows'ta anahtar oluşturma

```powershell
ssh-keygen -t ed25519 -C "windows-pc"
```

Sorulan sorularda Enter'a basarak varsayılanları kabul edebilirsiniz (isterseniz parola da koyabilirsiniz). Anahtarlar şu klasöre kaydedilir:

```
C:\Users\Kullanici\.ssh\id_ed25519       (özel anahtar)
C:\Users\Kullanici\.ssh\id_ed25519.pub   (genel/public anahtar)
```

### 2. Genel anahtarı Fedora'ya kopyalama

Windows'ta `ssh-copy-id` doğrudan bulunmayabilir; PowerShell'de şu yöntemi kullanabilirsiniz:

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh kullaniciadi@192.168.1.50 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

Eğer Git Bash veya WSL kullanıyorsanız `ssh-copy-id` de çalışır:

```bash
ssh-copy-id kullaniciadi@192.168.1.50
```

### 3. Fedora tarafında izinleri kontrol etme (gerekirse)

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### 4. Test

```powershell
ssh kullaniciadi@192.168.1.50
```

Artık şifre sormadan doğrudan bağlanmalı.

## 1.8 SSH Config Dosyası ile Kısayol Oluşturma

Her seferinde uzun komut yazmamak için Windows'ta `C:\Users\Kullanici\.ssh\config` dosyası oluşturun (yoksa yeni oluşturun, uzantısız):

```
Host fedorapc
    HostName 192.168.1.50
    User kullaniciadi
    Port 22
    IdentityFile ~/.ssh/id_ed25519
```

Bundan sonra bağlanmak için sadece şunu yazmanız yeterli:

```powershell
ssh fedorapc
```

SCP ile de kullanılabilir:

```powershell
scp dosya.txt fedorapc:/home/kullaniciadi/
```

## 1.9 Rsync (Opsiyonel, Daha Gelişmiş Dosya Senkronizasyonu)

Eğer Windows'ta WSL veya Git Bash gibi bir Linux ortamı varsa, `rsync` ile fark bazlı (delta) dosya aktarımı yapılabilir:

```bash
rsync -avz --progress ./proje/ kullaniciadi@192.168.1.50:/home/kullaniciadi/proje/
```

- `-a` : arşiv modu (izinler, zaman damgaları korunur)
- `-v` : ayrıntılı çıktı
- `-z` : sıkıştırarak aktarım
- `--progress` : ilerleme çubuğu

## 1.10 Sık Karşılaşılan Sorunlar

| Sorun | Çözüm |
|---|---|
| `Connection refused` | Fedora'da sshd servisi çalışmıyor olabilir → `sudo systemctl start sshd` |
| `Connection timed out` | Firewall engelliyor olabilir → `firewall-cmd` ile port açın, ya da farklı ağdasınız |
| `Permission denied (publickey,password)` | Kullanıcı adı/şifre yanlış ya da SSH ayarlarında şifre girişi kapalı |
| `Host key verification failed` | Fedora yeniden kurulduysa eski key değişmiştir → `ssh-keygen -R 192.168.1.50` ile eski kaydı silin |
| IP adresi sürekli değişiyor | Router'da Fedora'ya statik IP (DHCP reservation) atayın |

---

# BÖLÜM 2: Windows'tan Termux (Android) Bağlantısı

## 2.1 Termux Kurulumu (Android Tarafı)

> Önemli: Google Play Store'daki Termux sürümü güncellenmiyor/bozuk çalışabiliyor. **F-Droid** üzerinden kurulması önerilir.

1. F-Droid uygulamasını Android cihaza kurun (F-Droid resmi sitesinden apk indirilir).
2. F-Droid içinden "Termux" uygulamasını arayıp kurun.
3. Termux'u açın.

## 2.2 Termux'ta OpenSSH Kurulumu

Termux açıldıktan sonra önce paket listesini güncelleyin:

```bash
pkg update && pkg upgrade -y
```

OpenSSH paketini kurun:

```bash
pkg install openssh -y
```

## 2.3 Termux'ta SSH Sunucusunu Başlatma

```bash
sshd
```

Bu komut arka planda SSH sunucusunu başlatır (çıktı vermez, normaldir). Termux'ta SSH varsayılan portu **8022**'dir (22 değil, çünkü Android'de root olmayan uygulamalar 1024 altı portları kullanamaz).

Sunucunun çalıştığını kontrol etmek için:

```bash
pgrep sshd
```

Bir sayı (process ID) dönerse çalışıyordur.

## 2.4 Termux Kullanıcı Şifresi Belirleme

Termux'ta varsayılan olarak şifre yoktur, SSH ile bağlanabilmek için şifre belirlemeniz gerekir:

```bash
passwd
```

İstenen yeni şifreyi girin (Termux şifre yazarken de ekranda karakter göstermez).

## 2.5 Termux Kullanıcı Adı ve IP Adresini Öğrenme

Kullanıcı adı öğrenme:

```bash
whoami
```

(Genelde çıktı olarak size bir kullanıcı adı verir, örn. `u0_a123`)

Telefonun yerel ağ IP adresini öğrenme:

```bash
ifconfig
```

`ifconfig` yoksa önce kurun:

```bash
pkg install net-tools -y
```

veya alternatif olarak:

```bash
ip addr
```

`wlan0` bölümündeki `inet` satırında görünen adres (örn. `192.168.1.30`) kullanılacak IP'dir.

> Not: Telefon ve Windows PC **aynı Wi-Fi ağına** bağlı olmalıdır.

## 2.6 Windows'tan Termux'a SSH ile Bağlanma

Windows PowerShell veya CMD'de:

```powershell
ssh -p 8022 u0_a123@192.168.1.30
```

- `-p 8022` → Termux'un varsayılan SSH portu (mutlaka belirtilmeli, aksi halde bağlanamaz)
- `u0_a123` → Termux `whoami` çıktısındaki kullanıcı adı
- `192.168.1.30` → Telefonun yerel IP adresi

İlk bağlantıda host key onayı çıkar, `yes` yazıp devam edin, ardından `passwd` ile belirlediğiniz şifreyi girin.

## 2.7 Bağlantıdan Çıkma

Termux SSH oturumundan çıkmak için, Fedora'daki gibi:

```bash
exit
```

veya

```
Ctrl + D
```

Telefon tarafında SSH sunucusunu tamamen durdurmak isterseniz Termux uygulamasında:

```bash
pkill sshd
```

## 2.8 Windows'tan Termux'a Dosya Gönderme (SCP)

```powershell
scp -P 8022 C:\Users\Kullanici\Desktop\dosya.txt u0_a123@192.168.1.30:/data/data/com.termux/files/home/
```

> Termux'ta ev dizini genelde `~` ile kısaltılabilir ama SCP hedefinde tam yol vermek daha güvenlidir. Kısa yol olarak sadece `~/` de deneyebilirsiniz:

```powershell
scp -P 8022 dosya.txt u0_a123@192.168.1.30:~/
```

## 2.9 Termux'tan Windows'a Dosya İndirme (SCP)

```powershell
scp -P 8022 u0_a123@192.168.1.30:~/dosya.txt C:\Users\Kullanici\Desktop\
```

## 2.10 Klasör Transferi (Termux ile)

Windows'tan telefona klasör gönderme:

```powershell
scp -r -P 8022 C:\Users\Kullanici\Desktop\proje u0_a123@192.168.1.30:~/
```

Telefondan Windows'a klasör indirme:

```powershell
scp -r -P 8022 u0_a123@192.168.1.30:~/proje C:\Users\Kullanici\Desktop\
```

## 2.11 SFTP ile Termux'a Bağlanma (Alternatif)

```powershell
sftp -P 8022 u0_a123@192.168.1.30
```

Aynı SFTP komutları (Bölüm 1.6'daki tablo) burada da geçerlidir: `get`, `put`, `ls`, `lls`, `cd`, `lcd`, `exit`.

## 2.12 SSH Anahtarı ile Termux'a Şifresiz Giriş

### 1. Windows'ta anahtar oluşturma (zaten Bölüm 1.7'de oluşturduysanız aynısını kullanabilirsiniz)

```powershell
ssh-keygen -t ed25519 -C "windows-to-termux"
```

### 2. Genel anahtarı Termux'a kopyalama

Windows PowerShell'den:

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p 8022 u0_a123@192.168.1.30 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

### 3. Termux tarafında izinleri ayarlama

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### 4. Test

```powershell
ssh -p 8022 u0_a123@192.168.1.30
```

## 2.13 SSH Config ile Termux için Kısayol

`C:\Users\Kullanici\.ssh\config` dosyasına ekleyin:

```
Host telefon
    HostName 192.168.1.30
    User u0_a123
    Port 8022
    IdentityFile ~/.ssh/id_ed25519
```

Kullanımı:

```powershell
ssh telefon
scp dosya.txt telefon:~/
```

## 2.14 Termux'un Arka Planda Kalıcı Çalışması İçin Notlar

- Android, pil optimizasyonu nedeniyle arka plandaki Termux'u kapatabilir. Telefon ayarlarından Termux uygulaması için **"Pil optimizasyonu yok / Kısıtlama yok"** seçeneğini işaretleyin.
- Termux uygulamasını tamamen kapatmayın (son uygulamalar listesinden kaydırıp kapatmak SSH sunucusunu da durdurabilir); bildirim çubuğunda Termux servis bildirimi görünüyorsa arka planda aktif demektir.
- İsteğe bağlı: `termux-wake-lock` komutu ile CPU'nun uykuya geçmesini engelleyebilirsiniz (pil tüketimini artırır):

```bash
termux-wake-lock
```

Kapatmak için:

```bash
termux-wake-unlock
```

- Her Termux yeniden açılışında `sshd` komutunu tekrar çalıştırmanız gerekir (otomatik başlamaz). Otomatikleştirmek isterseniz `Termux:Boot` eklentisini kurup açılışta çalışacak bir script tanımlayabilirsiniz.

## 2.15 Sık Karşılaşılan Termux SSH Sorunları

| Sorun | Çözüm |
|---|---|
| `Connection refused` | `sshd` komutu çalıştırılmamış olabilir, tekrar `sshd` yazın |
| `Connection timed out` | Telefon ve PC aynı Wi-Fi ağında değil, ya da router cihazlar arası izolasyonu (AP isolation) açık |
| `Permission denied` | `passwd` ile şifre belirlemediniz veya yanlış kullanıcı adı giriyorsunuz (`whoami` ile kontrol edin) |
| Bağlantı port hatası veriyor | `-p 8022` parametresini unutmayın, Termux 22 değil 8022 kullanır |
| Telefon uyuyunca bağlantı kopuyor | Pil optimizasyonunu kapatın, gerekirse `termux-wake-lock` kullanın |

---

# Ekstra: Genel SSH/SCP Komut Özet Tablosu

| İşlem | Komut |
|---|---|
| Fedora'ya bağlan | `ssh kullanici@ip` |
| Termux'a bağlan | `ssh -p 8022 kullanici@ip` |
| Farklı portla bağlan | `ssh -p PORT kullanici@ip` |
| Bağlantıdan çık | `exit` veya `Ctrl+D` |
| Dosya gönder (SCP) | `scp dosya kullanici@ip:/hedef/yol/` |
| Dosya al (SCP) | `scp kullanici@ip:/hedef/dosya ./` |
| Klasör gönder/al | `scp -r ...` |
| Termux'a SCP (port belirterek) | `scp -P 8022 dosya kullanici@ip:~/` |
| SFTP başlat | `sftp kullanici@ip` |
| SSH anahtarı oluştur | `ssh-keygen -t ed25519` |
| Anahtarı sunucuya gönder | `ssh-copy-id kullanici@ip` (veya `type ... | ssh ... "cat >> ..."`) |
| Config ile kısa bağlan | `ssh takmaisim` (config dosyası tanımlıysa) |

---

**Not:** Yukarıdaki tüm `192.168.1.50`, `192.168.1.30`, `kullaniciadi`, `u0_a123` gibi değerler örnektir; kendi cihazlarınızdaki gerçek IP adresi ve kullanıcı adlarıyla değiştirmeniz gerekir.
