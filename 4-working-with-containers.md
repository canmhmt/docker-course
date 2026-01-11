# Docker Konteynerları

### 1. Konteynır: En Kısa Özet (TLDR)

Konteynır, bir imajın **çalışan bir kopyasıdır (instance).**

* **İlişki:** İmaj bir "yemek tarifidir", konteynır ise o tarife göre pişmiş "yemektir".
* **Çoğaltılabilirlik:** Tek bir imajdan (örneğin Redis imajı), birbirinden tamamen bağımsız 10 tane Redis konteynırı başlatabilirsin.

### 2. İmajlar ve Konteynırlar Arasındaki Yazma Farkı

Buradaki en kritik teknik detay şudur: **İmajlar salt okunurdur (read-only).**

* Bir imajdan konteynır başlattığında, Docker imajın üzerine ince bir **"Yazılabilir Katman" (Writable Layer)** ekler.
* Konteynır içinde yaptığın tüm değişiklikler (dosya oluşturma, log yazma vb.) bu ince katmana yazılır. Alttaki ana imaj asla değişmez.

---

### 3. Konteynır vs. Sanal Makine (VM)

Konteynırları VM'lerden ayıran temel farklar şunlardır:

| Özellik | Konteynır | Sanal Makine (VM) |
| --- | --- | --- |
| **Boyut** | Çok küçük (MB seviyesi) | Büyük (GB seviyesi) |
| **Hız** | Saniyeler içinde açılır | Dakikalar içinde açılır |
| **Taşınabilirlik** | Çok yüksek (Her yerde aynı çalışır) | Daha hantal |
| **Yaşam Döngüsü** | Geçici ve Statüsüz (Ephemeral) | Uzun ömürlü ve Kalıcı |

---

### 4. Konteynır Felsefesi: Üç Altın Kural

#### A. Değişmezlik (Immutability)

Konteynırlar "tamir edilmek" için değil, **"yenisiyle değiştirilmek"** için tasarlanmıştır. Eğer bir konteynır hata veriyorsa, içine girip kodu düzeltmezsin. Kodu düzeltip yeni bir imaj basar ve eski konteynırı silip yenisini ayağa kaldırırsın.

#### B. Tek Bir Süreç (Single Process)

Bir konteynır sadece **bir ana iş** yapmalıdır. İçine hem veritabanı hem web server koyulmaz.

* Web server için ayrı konteynır.
* Veritabanı için ayrı konteynır.
Bu yaklaşım, mikroservis mimarisinin temelidir.

#### C. Geçicilik (Ephemerality)

Konteynırın her an ölebileceğini varsayarak tasarım yapmalısın. İçindeki veriler uçup gidebilir (stateless). Eğer veri kalıcı olacaksa (veritabanı gibi), bunu konteynırın dışında (Volumes) saklamalısın.

---

### 5. OCI (Open Container Initiative) Standardı

Metinde geçen OCI vurgusu çok önemlidir. Docker, konteynırların nasıl olması gerektiğini belirleyen bu standartlara uyar. Bu sayede:

* Docker ile build ettiğin bir imajı **Podman** veya **Containerd** ile de çalıştırabilirsin.
* Yazdığın komutların ve mantığın büyük çoğunluğu diğer platformlarda da geçerli olur.

---

### 6. Kritik Tartışma: Ortak Çekirdek (Shared Kernel) Güvenli mi?

Eskiden konteynerlerin en çok eleştirilen noktası, hepsinin aynı işletim sistemi çekirdeğini (kernel) paylaşmasıydı.

* **Risk:** Eğer bir konteyner çekirdekteki bir açığı sömürürse, aynı makinedeki diğer tüm konteynerlere zarar verebilir.
* **Güncel Durum:** Metin, bu endişenin artık yersiz olduğunu belirtiyor. Çünkü güncel Docker ve konteyner platformları şu ileri seviye güvenlik araçlarıyla donatılmıştır:
* **SELinux / AppArmor:** Uygulamanın neye erişip neye erişemeyeceğini belirleyen sıkı kurallar.
* **Seccomp:** Konteynerin işletim sistemi çekirdeğine hangi komutları gönderebileceğini kısıtlar.
* **Capabilities:** Root yetkilerini parçalara ayırır, böylece konteyner "root" olsa bile her şeyi yapamaz.
* **Vulnerability Scanning:** Önceki bölümlerde gördüğümüz **Docker Scout** gibi araçlar sayesinde, imajın içindeki her türlü açık daha yayınlanmadan tespit edilir.

### Sonuç

Konteynerler artık sadece bir "alternatif" değil; modern yazılım dünyasının **standart çözümüdür.** Verimli, hızlı ve modern araçlarla desteklendiğinde VM'lerden daha güvenli hale getirilebilirler.

---

## Docker Yazılabilir Katman (R/W Layer) Mantığı

### 1. İmaj ve Konteynır Arasındaki "Paylaşım" Modeli

Bir imajdan 100 tane konteynır başlatsanız bile, diskte 100 tane imaj yer kaplamaz. Docker, **Union File System** denilen bir teknoloji kullanarak imajı ve konteynırı birbirinden ayırır:

* **İmaj (Read-Only):** En alttaki katman kümesidir. Asla değişmez, silinmez veya üzerine yazılamaz. Tüm konteynırlar bu katmanı ortak olarak "okur".
* **Konteynır (Read-Write Layer):** Bir konteynır başlattığınız anda, Docker imajın en üstüne çok ince bir **"yazılabilir katman"** ekler.

---

### 2. Yazılabilir Katman (R/W Layer) Nasıl Çalışır?

Konteynırın içinde bir dosya oluşturduğunuzda veya bir ayarı değiştirdiğinizde olanlar şunlardır:

1. **Okuma:** Konteynır bir dosyaya erişmek istediğinde, önce kendi R/W katmanına bakar. Orada yoksa alt taraftaki paylaşılan imaj katmanlarına iner ve dosyayı oradan okur.
2. **Yazma (Copy-on-Write):** Eğer imajda var olan bir dosyayı değiştirmek isterseniz, Docker o dosyayı imajdan alır, kendi R/W katmanına kopyalar ve değişikliği orada yapar. İmajın içindeki orijinal dosya olduğu gibi kalır.
3. **İzolasyon:** Konteynır A'nın yaptığı bir değişiklik, sadece Konteynır A'nın R/W katmanında durur. Konteynır B bunu göremez.

---

### 3. Konteynır Durunca veya Silinince Ne Olur?

Bu katmanın yaşam döngüsü, konteynırın durumuyla doğrudan bağlantılıdır:

* **Stop (Durdurma):** Konteynırı durdurduğunuzda R/W katmanı **silinmez.** Konteynırı tekrar `start` ettiğinizde tüm değişiklikleriniz (yazdığınız loglar, oluşturduğunuz dosyalar) geri gelir.
* **Delete/Remove (Silme):** Konteynırı sildiğinizde (`docker rm`), Docker ona ait olan o özel R/W katmanını da **tamamen siler.** > **Önemli Not:** Konteynır silindiğinde verilerin kaybolmasının sebebi budur. Bu yüzden veritabanı gibi kalıcı olması gereken verileri bu geçici R/W katmanında değil, **Volumes (Hacimler)** dediğimiz harici alanlarda saklarız.

---

### 4. Neden Bu Sistem Çok Verimli?

* **Hız:** Yeni bir konteynır başlatmak, koca bir işletim sistemini kopyalamak demek değildir; sadece boş ve ince bir yazılabilir katman oluşturmaktır. Bu işlem milisaniyeler sürer.
* **Alan Tasarrufu:** 1 GB'lık bir imajdan 10 tane konteynır çalıştırdığınızda, diskte hala yaklaşık 1 GB yer kaplanır. Ekstra harcanan alan sadece her konteynırın kendi içinde oluşturduğu küçük veriler kadardır.

---

- Docker'ın çalışıp çalışmadığını kontrol etmek, sadece bir "merhaba" demek değildir; aslında **İstemci (Client)** ile **Sunucu (Server)** arasındaki iletişimin sağlıklı olduğunu doğrulamaktır. Metindeki teknik detayları ve olası sorun giderme adımlarını eksiksiz inceleyelim:

### 1. `docker version` Komutu Neden Önemli?

Bu komut, Docker'ın iki ana parçasını da sorgular:

* **Client (İstemci):** Terminale yazdığın komutları alan araç.
* **Server (Engine/Daemon):** Arka planda konteynırları asıl yöneten motor.

Eğer her iki bölümden de (versiyon numarası, OS/Arch gibi) yanıt alıyorsan, Docker sistemi "sağlıklı" demektir.

### 2. "Server" Hatası Alıyorsan Ne Demektir?

Eğer sadece `Client` bilgilerini görüyor ama `Server` kısmında hata alıyorsan, bu genellikle şu iki sorundan biridir:

1. **Yetki Sorunu (Linux için):** Docker motoru, `/var/run/docker.sock` adlı özel bir "kapı" (Unix socket) üzerinden konuşur. Bu kapıya sadece "ayrıcalıklı" kullanıcılar dokunabilir.
2. **Motor Çalışmıyor:** Docker arka plan süreci (Daemon) kapalıdır.

---

### 3. Linux'ta Yetki Sorununu Çözmek

Linux kullanıyorsan ve her seferinde `sudo` yazmak istemiyorsan, kullanıcını `docker` grubuna eklemelisin:

* **Komut:** `sudo usermod -aG docker <kullanıcı_adın>`
* **Mantık:** Bu işlem, senin kullanıcına socket açma anahtarı verir.
* **Not:** Ayarın aktif olması için terminali kapatıp açmalı veya oturumu yeniden başlatmalısın.

---

### 4. Daemon Durumunu Kontrol Etmek

Eğer yetkin varsa ama hala hata alıyorsan, motorun çalışıp çalışmadığına bakman gerekir:

* **Systemd kullanan sistemler (Modern Linux):**
`systemctl is-active docker` (Yanıt "active" olmalı).
* **Eski sistemler:**
`service docker status`.

Eğer "inactive" (kapalı) ise, `sudo systemctl start docker` ile motoru ateşleyebilirsin.

---

### Teknik Detay: Unix Socket vs Network API

- `/var/run/docker.sock` detayı çok kritiktir.

* **Local (Yerel):** Docker varsayılan olarak bu dosya üzerinden haberleşir. Bu en güvenli yoldur.
* **Remote (Uzak):** İstersen Docker'ı bir IP adresi ve Port üzerinden ağa açabilirsin (Örn: `tcp://192.168.1.10:2375`). Ancak bu, doğru güvenlik önlemleri alınmazsa bilgisayarını dış dünyaya tamamen korumasız bırakabilir.

---

### Özet: Hızlı Arıza Tespit Tablosu

| Aldığın Hata | Olası Neden | Çözüm |
| --- | --- | --- |
| `Cannot connect to the Docker daemon` | Yetki eksikliği | `sudo` kullan veya gruba ekle |
| `Server: <hata>` | Docker kapalı | `systemctl start docker` |
| `OS/Arch mismatch` | Yanlış mimari | İmajın senin işlemcine uygunluğunu kontrol et |

---

## Docker Run Komutu

### 1. Komutun Anatomisi: Her Bayrak Ne İş Yapar?

`$ docker run -d --name webserver -p 5005:8080 nigelpoulton/ddd-book:web0.1`

* **`docker run`**: "Yeni bir konteynır yarat ve onu başlat" talimatıdır.
* **`-d` (Detached)**: Konteynırın arka planda (daemon olarak) çalışmasını sağlar. Terminalini kilitlemez, sen başka komutlar yazmaya devam edebilirsin.
* **`--name webserver`**: Konteynıra senin belirlediğin bir isim verir. Eğer bunu yazmazsan, Docker rastgele (genellikle komik) bir isim atar.
* **`-p 5005:8080` (Port Mapping)**: Dış dünyayı konteynıra bağlayan köprüdür. Senin bilgisayarındaki (host) **5005** portuna gelen istekleri, konteynırın içindeki **8080** portuna yönlendirir.
* **`nigelpoulton/ddd-book:web0.1`**: Konteynırın hangi "tarif defterinden" (imajdan) üretileceğini belirtir.

---

### 2. Arka Planda Neler Oluyor? (Sahne Arkası)

Sen "Enter" tuşuna bastığında şu zincirleme reaksiyon gerçekleşir:

1. **API İsteği**: Docker İstemcisi (CLI), komutu bir API isteğine dönüştürüp Docker Daemon'a (motor) gönderir.
2. **İmaj Kontrolü**: Daemon, yerel deposuna bakar. İmajı bulamazsa Docker Hub'a gider, imajı bulur ve katman katman indirir (Pulling).
3. **Konteynır Oluşturma**: Daemon, elindeki imajla birlikte **containerd** birimine gider.
4. **Düşük Seviye İşlem**: **containerd**, işletim sistemi seviyesinde konteynırı oluşturması için **runc** aracını tetikler.
5. **Uygulama Başlatma**: **runc**, konteynırı izole bir alanda (namespaces ve cgroups kullanarak) yaratır ve içindeki uygulamayı (`node ./app.js`) başlatır.

---

### 3. Doğrulama: Her Şey Yolunda mı?

Konteynırın durumunu kontrol etmek için iki temel komut kullanılır:

* **`docker images`**: İmajın başarıyla indirilip indirilmediğini ve diskte ne kadar yer kapladığını doğrular.
* **`docker ps`**: Şu an "canlı" olan konteynırları listeler.
* **STATUS**: `Up 2 mins` ifadesi konteynırın sapa sağlam çalıştığını gösterir.
* **PORTS**: Port eşleşmesinin (`0.0.0.0:5005->8080/tcp`) doğru yapıldığını buradan teyit edersin.

---

### 4. Uygulamaya Erişim

Tarayıcını açıp `localhost:5005` yazdığında şunlar olur:

1. İstek senin bilgisayarının 5005 portuna çarpar.
2. Docker'ın kurduğu ağ köprüsü (bridge) bu isteği yakalar.
3. İsteği konteynırın içindeki 8080 portuna, yani Node.js uygulamasına iletir.
4. Uygulama cevap verir ve sen web sayfasını görürsün.

### Kritik Özet

- Bu işlem, Docker'ın neden bu kadar güçlü olduğunu gösterir: **Altyapı (Networking, Storage, Process Isolation)** dakikalarca uğraşmak yerine tek bir satır komutla otomatik olarak kuruldu.

---

