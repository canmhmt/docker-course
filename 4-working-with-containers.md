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

## Konteyner Bir Uygulamayı Nasıl Çalıştırır?

- Bir konteynırın içindeki uygulamanın kendi kendine nasıl başladığını anlamak, Docker'ın en kritik "mantık" bölümlerinden biridir. Bir konteynır aslında sadece içinde çalışan **bir ana süreçten (process)** ibarettir; o süreç bittiğinde konteynır da biter.

- İşte Docker'ın bu süreci başlatmak için kullandığı üç yöntem ve aralarındaki farklar:

### 1. Giriş Noktaları: Entrypoint ve Cmd

- Bir imajın içinde, o imajdan bir konteynır yaratıldığında neyin çalışacağını söyleyen iki temel talimat (metadata) bulunur.

* **Entrypoint (Giriş Noktası):** İmajın "asıl amacı"dır. Değiştirilmesi (override edilmesi) zordur. Genellikle uygulamanın ana çalıştırılabilir dosyasını tutar.
* **Cmd (Varsayılan Komut):** Uygulamaya geçilecek varsayılan parametrelerdir. Eğer kullanıcı terminalde başka bir komut yazarsa, `Cmd` tamamen yok sayılır.

### 2. Kritik Fark: Ezme mi, Ekleme mi?

Metindeki en önemli teknik ayrım şudur: Eğer terminalden (`CLI`) bir komut gönderirseniz:

* **İmajda `Entrypoint` varsa:** Yazdığınız komut, `Entrypoint`'in yanına bir **argüman** olarak eklenir. (Örneğin: `Entrypoint` "node" ise ve siz "index.js" yazarsanız, komut "node index.js" olur).
* **İmajda `Cmd` varsa:** Yazdığınız komut, `Cmd`'yi **tamamen siler** ve yerine geçer.

| Belirleme Yeri | Tip | CLI Argümanı ile Ne Olur? |
| --- | --- | --- |
| **İmaj İçinde** | `Entrypoint` | CLI argümanı sona eklenir. |
| **İmaj İçinde** | `Cmd` | CLI argümanı `Cmd`'yi ezer/siler. |
| **CLI (Terminal)** | `docker run ... <komut>` | İmajda talimat yoksa bu çalışır. |

---

### 3. Uygulamalı Örnekler

#### İmaj Meta-verisini İncelemek

`docker inspect` komutu ile imajın "tarif defterine" bakıp neyi çalıştırmak üzere programlandığını görebiliriz:

```bash
docker inspect nigelpoulton/ddd-book:web0.1 | grep Entrypoint -A 3

```

Bu komutun çıktısında `node ./app.js` görüyorsak, bu konteynır ayağa kalktığı an Node.js motorunu kullanarak web uygulamasını başlatacak demektir.

#### CLI ile Komut Gönderme (`--rm` ve `sleep`)

Bazen imajın varsayılan işini yapmasını değil, bizim istediğimiz bir şeyi yapmasını isteriz:

```bash
docker run --rm -d alpine sleep 60

```

* **`alpine`**: İmaj adı.
* **`sleep 60`**: Bu bir CLI argümanıdır. Alpine imajında genellikle `Entrypoint` yoktur, sadece `Cmd` (shell) vardır. Biz `sleep 60` yazarak varsayılanı ezeriz ve konteynırın 60 saniye boyunca hiçbir şey yapmadan beklemesini sağlarız.
* **`--rm`**: Bu çok pratik bir bayraktır. Konteynırın işi bittiğinde (60 saniye dolunca) onu otomatik olarak çöpe atar. Sistemin kirlenmesini engeller.

---

### 4. Özet: Konteynırın Yaşam Döngüsü

Bir konteynır, içinde çalışan **ana süreç (Entrypoint veya Cmd ile başlayan uygulama)** canlı olduğu sürece ayaktadır.

* Eğer web sunucusu bir hata verip kapanırsa, konteynır durur.
* Eğer `sleep 60` süresi dolarsa, ana süreç biter ve konteynır durur.

--- 

## Çalışan bir konteynera bağlanmak

Çalışan bir konteynerin içine girmek, sanki başka bir sunucuya SSH ile bağlanmak gibidir. Ancak Docker bunu çok daha hızlı ve hafif bir yolla, `docker exec` komutuyla yapar. Bu işlemin mantığını eksiksiz inceleyelim:

### 1. `docker exec` Nedir?

Bu komut, **zaten çalışmakta olan** bir konteynerin içinde ek bir süreç (process) başlatmanızı sağlar. İki farklı şekilde kullanılabilir:

* **Interactive (Etkileşimli):** Terminalinizi konteynerin içindeki bir programa (genellikle bir shell) bağlar.
* **Remote Execution (Uzaktan Çalıştırma):** Konteynerin içine girmeden dışarıdan bir komut gönderirsiniz, cevabı alır ve terminalinize dönersiniz (Örn: `docker exec webserver ls`).

---

### 2. Etkileşimli Bağlantı: `-it` Bayrağının Sırrı

Metindeki şu komutu parçalayalım:
`$ docker exec -it webserver sh`

* **`-i` (interactive):** Konteyner ile aranızdaki iletişimi açık tutar (girdi göndermenizi sağlar).
* **`-t` (tty):** Size gerçek bir terminal ekranı (pseudo-TTY) tahsis eder. Promptun (imlecin) değişmesini sağlayan budur.
* **`webserver`:** Hangi konteynerin içine gireceğinizi belirtir.
* **`sh`:** Konteynerin içinde hangi programı çalıştıracağınızı söyler. `sh` (Shell), Linux dünyasındaki en temel komut satırı arayüzüdür.

---

### 3. Konteyner Neden "Eksik" Gibi Hissettirir?

Konteynerin içine girdiğinizde `ls` komutu çalışırken `vim` veya `nano` gibi komutların çalışmadığını görebilirsiniz. Bunun çok önemli bir sebebi vardır: **Hafiflik.**

* **İmaj Optimizasyonu:** Bir Docker imajı (özellikle Alpine tabanlı olanlar), sadece uygulamanın çalışması için gereken **minimum** dosyaları içerir.
* **Güvenlik:** İçinde editör (vim), ağ araçları (curl, ping) veya derleyiciler (gcc) bulunmayan bir konteyner, saldırganlar için çok daha zor bir hedeftir. Eğer bir araç imajda yoksa, o konteynerde o aracı kullanamazsınız.

---

### 4. Kritik Uyarı: Konteynerin İçinde Dosya Değiştirmek

Metindeki örnekte `app.js` dosyasını görmenize rağmen onu düzenleyememeniz aslında Docker felsefesine uygundur:

* **Geçicilik:** Eğer konteynerin içine girip bir dosyayı manuel olarak değiştirirseniz (örneğin `vim` yüklü olsaydı ve kodu değiştirseydiniz), konteyner silindiğinde bu değişiklik **yok olur.**
* **Doğru Yöntem:** Değişikliği kendi bilgisayarınızdaki kod dosyasında yapmalı, yeni bir imaj build etmeli ve konteynerinizi güncellenmiş imajla yeniden başlatmalısınız.

---

### 5. PID 1: Konteynerin Kalbi

Linux sistemlerde **PID 1** (Process ID 1), sistemin ilk ve en yetkili sürecidir. Docker dünyasında ise PID 1, konteynerin başlatılma sebebidir (yani `Entrypoint` veya `Cmd` ile belirttiğimiz uygulama).

* **Bağımlılık:** Konteyner, sadece PID 1 sürecini hayatta tutmak için var olur.
* **Ölüm Fermanı:** Eğer PID 1 (örneğin Node.js uygulamanız) çökerse veya durursa, Docker "yapacak iş kalmadı" diyerek konteyneri anında kapatır.

---

### 6. Süreç Analizi: İçeride Neler Dönüyor?

| PID | Komut | Durumu |
| --- | --- | --- |
| **1** | `node ./app.js` | **Ana Süreç:** Bu durursa konteyner ölür. |
| **13** | `sh` | **Ek Süreç:** Senin `docker exec -it` ile açtığın tüneldir. Sen `exit` dersen bu süreç biter ama konteyner (PID 1 sayesinde) çalışmaya devam eder. |
| **22** | `ps` | **Geçici Süreç:** Sadece o anki listeyi göstermek için doğar ve işi bitince (milisaniyeler içinde) yok olur. |

---

### Özet: İpucu ve Notlar

* **Çıkış Yapmak:** Konteynerin içindeki shell'den çıkmak için `exit` yazabilir veya `Ctrl + D` tuşlarına basabilirsiniz. Bu işlem konteyneri durdurmaz, sadece sizin bağlantınızı koparır.
* **sh vs bash:** Bazı imajlarda `sh` yerine daha gelişmiş bir shell olan `bash` bulunur. Eğer `sh` çok kısıtlı gelirse `docker exec -it webserver bash` komutunu deneyebilirsiniz (tabii imajda yüklüyse).

---

## Docker Inspect Komutu

### 1. `docker inspect`: Konteynerin "Röntgenini" Çekmek

`docker inspect` komutu, Docker Daemon'ın o konteyner veya imaj hakkında bildiği **her şeyi** (JSON formatında) size sunar. Sadece çalışan süreci değil, altyapı ayarlarını da görmenizi sağlar.

Metindeki örnekte öne çıkan kritik bilgiler şunlardır:

* **State (Durum):** Konteynerin o anki yaşam belirtisidir. `running` (çalışıyor) bilgisini buradan teyit ederiz.
* **PortBindings:** Dış dünyadaki hangi kapının (HostPort: `5005`), konteynerin içindeki hangi kapıya (ContainerPort: `8080`) bağlı olduğunu net bir şekilde gösterir.
* **RestartPolicy:** Konteyner çökerse Docker'ın ne yapacağını söyler. "no" olması, konteyner durursa Docker'ın onu otomatik olarak tekrar başlatmayacağı anlamına gelir.
* **Image & WorkingDir:** Konteynerin hangi temelden yükseldiğini ve dosya sisteminde hangi klasörü (`/src`) merkez aldığını belirtir.

---

### 2. Ayarların Kaynağı: İmajdan Konteynere Miras

Metindeki en önemli teknik vurgu **kalıtımdır (inheritance).**

Konteyneri inspect ettiğinizde gördüğünüz `Entrypoint` (başlangıç komutu olan `node ./app.js`), aslında konteynere özel bir ayar değildir. Konteyner bu ayarı, yaratıldığı **imajdan miras almıştır.**

**Bunu nasıl kanıtlarız?**

1. Önce konteyneri inceleyin: `docker inspect webserver`
2. Sonra imajı inceleyin: `docker inspect nigelpoulton/ddd-book:web0.1`

İkisinde de `Entrypoint` kısmının aynı olduğunu göreceksiniz. Bu, Docker'ın "bir kez paketle, her yerde aynı ayarla çalıştır" mantığının teknik kanıtıdır.

---

### 3. Neden Inspect Komutunu Kullanmalıyız?

Hata ayıklama (debugging) sırasında şu soruların cevabını sadece `inspect` ile hızlıca bulabilirsiniz:

* "Bu konteynerin IP adresi ne?"
* "Hangi Volume'lar (diskler) bağlı?"
* "Hangi ortam değişkenleri (Environment Variables) tanımlanmış?"
* "Konteyner neden durdu? (ExitCode kaç?)"

---

## Konteyner Yaşam Döngüsü Kontrolü

### 1. Durdurma (`stop`) ve Yeniden Başlatma (`restart`)

Bir konteynırı `docker stop` ile durdurduğunda:

* **İşlem:** Docker, ana sürece (PID 1) bir kapatma sinyali gönderir. Eğer uygulama 10 saniye içinde kapanmazsa Docker onu zorla kapatır.
* **Görünürlük:** `docker ps` yazdığında konteynırı göremezsin; ancak `docker ps -a` (all) yazdığında `Exited` (Çıkış yaptı) durumunda orada beklediğini görürsün.
* **Veri Durumu:** Önceki adımda `vi` ile yaptığın o meşhur değişiklik **kaybolmaz.** Çünkü konteynırın "yazılabilir katmanı" (thin R/W layer) hala diskte durmaktadır.

### 2. Kritik Kanıt: Değişiklikler Neden Kalıyor?

Konteynırı `docker restart` ile tekrar ayağa kaldırdığında ve tarayıcını yenilediğinde veya `docker exec webserver cat views/home.pug` komutuyla dosyayı kontrol ettiğinde, yaptığın değişikliğin hala orada olduğunu görürsün.

* **Ders:** Konteynırı durdurup başlatmak veriyi silmez. Konteynırın "hafızası" (yazılabilir katmanı), konteynır objesi sistemden tamamen silinene kadar yaşamaya devam eder.

### 3. Silme (`rm`) ve Verinin Yok Oluşu

Her şeyi değiştiren komut `docker rm` komutudur.

* **İşlem:** `docker rm webserver -f` komutuyla konteynırı sildiğinde, Docker bu konteynıra ait olan o özel "yazılabilir katmanı" da çöpe atar.
* **Sonuç:** Konteynır artık `docker ps -a` listesinde bile yoktur. Eğer aynı imajdan tekrar bir konteynır başlatırsan (`docker run`), karşına orijinal (değiştirilmemiş) uygulama çıkar. Yaptığın tüm manuel değişiklikler sonsuza dek kaybolmuştur.

---

### Bölümün Teknik Özeti (Lifecycle)

| Komut | Konteynırın Durumu | Veri (R/W Layer) |
| --- | --- | --- |
| `docker stop` | Exited (Durdu) | Güvende (Diskte duruyor) |
| `docker restart` | Up (Çalışıyor) | Mevcut (Eski değişiklikler duruyor) |
| `docker rm -f` | Yok Edildi | **SİLİNDİ** (Geri getirilemez) |

### Önemli Kavram: Anti-Pattern

Metinde geçen **Anti-Pattern** vurgusu çok kritiktir.

* **Anti-Pattern:** Çalışan ama "yapılmaması gereken" yöntemdir.
* **Neden?** Konteynırlar "geçici" (ephemeral) tasarlanmıştır. Eğer bir konteynırı silince uygulamanın ayarları veya verileri gidiyorsa ve sen bunu istemiyorsan, yanlış yoldasın demektir. Gerçek dünyada veriler konteynırın içinde değil, **Volume** adı verilen kalıcı dış alanlarda saklanır.

---

### 4. Konteynere Yeniden Bağlanma (`attach`)

Durdurduğun veya `Ctrl+PQ` ile çıktığın bir konteynere geri dönmek için şu akış kullanılır:

1. **Durdurulmuşsa:** `docker restart ddd-ctr` (Süreci tekrar canlandırır).
2. **Bağlanmak için:** `docker attach ddd-ctr` (Terminalini çalışan o PID 1 sürecine tekrar bağlanır).

---

### 5. Docker Debug: "Zayıf" (Slim) İmajları Kurtarmak

Metnin sonunda değinilen **Docker Debug**, modern konteyner dünyasının en büyük sorunlarından birini çözer: **İçinde araç olmayan imajlar.**

Bildiğin üzere `alpine` veya `distroless` gibi "slim" imajlarda `ps`, `ls` ve hatta bazen `sh` bile bulunmaz. Bu konteynerler bozulduğunda içine girip bakamazsın.

* **Docker Debug** bir eklentidir ve konteynere dışarıdan "bir alet çantası" (kendi shell'i ve araçları olan bir katman) getirir.
* Konteyner ne kadar zayıf veya bozuk olursa olsun, `docker debug <container-id>` komutuyla içine sızıp hata ayıklama yapabilirsin.

---

### Özet: Kritik Bilgiler Tablosu

| Eylem | PID 1 Süreci | Konteyner Durumu |
| --- | --- | --- |
| `exit` komutu | Sonlanır | **Durur (Exited)** |
| `Ctrl + P + Q` | Çalışmaya devam eder | **Aktif (Up)** |
| `docker stop` | Kapatma sinyali alır | **Durur (Exited)** |
| `docker kill` | Zorla sonlandırılır | **Anında Durur** |
| `attach` | Konteynerın PID 1´ine bağlanmak |


--- 

## Docker Debug

---

### 1. Sorun Nedir? (Slim İmaj Paradoksu)

Güvenlik ve performans için "Slim" (zayıflatılmış) imajlar kullanmak "en iyi pratik" (best practice) olarak kabul edilir.

* **Avantajı:** İçinde `shell` (kabuk), `curl`, `vim` veya `ping` gibi araçlar olmadığı için hackerlar sisteme sızsa bile hareket edemezler.
* **Dezavantajı:** Bir sorun çıktığında **sen de** hareket edemezsin. Çünkü `docker exec` yapıp içeri girsen bile çalıştıracak bir komut bulamazsın.

### 2. Çözüm: Docker Debug Nasıl Çalışır?

Docker Debug, adeta bir "acil durum çantası" ile konteynerin yanına yaklaşan bir teknisyendir.

* **Mekanizma:** Konteynerin içine özel bir `/nix` klasörü bağlar (mount eder).
* **İzolasyon:** Bu `/nix` klasörü ve içindeki araçlar sadece senin hata ayıklama oturumunda (debug session) görünür. Konteynerin kendi uygulaması bu klasörü göremez.
* **Erişim:** Konteynerde hiç shell (`sh`, `bash`) olmasa bile, Docker Debug kendi shell'ini getirerek içeri girmeni sağlar.

---

### 3. Ön Hazırlık ve Gereksinimler

* **Lisans:** Bu özellik şu an için sadece **Pro, Team veya Business** lisansı olan kullanıcılara açıktır.
* **Kurulum:** Docker Desktop'ın güncel sürümlerinde (4.27+) varsayılan olarak gelir. `docker info` komutunu çalıştırıp `Plugins` altında `debug` satırını görerek kontrol edebilirsin.

---

### 4. Senaryo 1: Çalışan Bir Konteyneri Debug Etmek

Metindeki örnekte, içinde `ping`, `vim` veya `nslookup` olmayan çıplak bir `ubuntu` konteyneri (`ddd-ctr`) kullanılıyor.

#### Adım A: Başarısızlık Testi

Önce normal yolla (`docker attach`) bağlanıp komutları deniyoruz:
`root@d3c892ad0eb3:/# ping google.com` -> **HATA:** `command not found`.

#### Adım B: Docker Debug ile Giriş

`$ docker debug ddd-ctr` komutunu verdiğinde:

1. Seni `docker >` şeklinde özel bir prompt karşılar.
2. Artık `ping` ve `vim` gibi araçlar varsayılan olarak elinin altındadır.
3. **Kritik Fark:** Burada yaptığın dosya değişiklikleri (örn: `vim index.html`) **kalıcıdır.** Debug oturumunu kapatsan bile konteynerde bu değişiklik kalır.

#### Adım C: Eksik Aracı Yüklemek (`install`)

Diyelim ki `nslookup` komutuna ihtiyacın var ama debug kutusunda bile yok.

* **Komut:** `docker > install bind`
* **Kaynak:** Araç, NixOS paket deposundan (`search.nixos.org`) anında indirilir ve senin kişisel araç çantana eklenir. Bir sonraki debug oturumunda tekrar yüklemene gerek kalmaz.

---

### 5. Senaryo 2: Bir İmajı Debug Etmek (Sandbox Modu)

Sadece çalışan konteynerleri değil, durağan bir imajı da (`nigelpoulton/ddd-book:web0.1`) inceleyebilirsin.

* **Komut:** `$ docker debug nigelpoulton/ddd-book:web0.1`
* **Uyarı Mesajı:** *"This is a sandbox shell. All changes will not affect the actual image."*
* **Mantık:** Docker, bu imajdan geçici bir katman oluşturur. Sen dosya silip eklesen bile, debug oturumundan `exit` de diğin anda **her şey sıfırlanır.** İmajın orijinali asla bozulmaz.

---

### 6. `entrypoint` Analizi

Debug modundayken çok yetenekli bir komut daha vardır:
`docker > entrypoint --print`

* Bu komut, imajın meta verilerini tarar ve sana "Bu konteyner başladığında tam olarak hangi komutu çalıştıracak?" sorusunun cevabını verir (Örn: `node ./app.js`).

---

### Özet: Kalıcılık (Persistence) Kuralları

| Debug Hedefi | Dosya Değişiklikleri | Yüklenen Araçlar (`/nix`) |
| --- | --- | --- |
| **Çalışan Konteyner** | **Kalıcıdır** (Dikkat!) | Debug oturumunda kalır |
| **İmaj / Durmuş Konteyner** | **Silinir** (Geçicidir) | Debug oturumunda kalır |

---

## Yeniden Başlatma Politikaları (Restart Policies)

### 1. Dört Temel Politika

Docker, konteyner başına uygulayabileceğiniz şu 4 politikayı destekler:

1. **no (Varsayılan):** Konteyner durursa asla otomatik başlatılmaz.
2. **on-failure:** Sadece konteyner bir hata ile (Non-zero exit code) kapanırsa yeniden başlatılır.
3. **always:** Konteyner nasıl kapanırsa kapansın (hata veya normal çıkış) her zaman yeniden başlatılır.
4. **unless-stopped:** `always` gibidir, ancak eğer konteyneri elle (`docker stop`) durdurursanız, Docker Daemon (motoru) yeniden başladığında bu konteyner kapalı kalmaya devam eder.

---

### 2. Senaryo Tablosu

Hangi politikanın hangi durumda nasıl tepki verdiğini gösteren tabloyu aşağıda netleştirdim. (**Y**: Yeniden Başlatır, **N**: Başlatmaz)

| Politika | Hatalı Çıkış (Non-zero exit code) | Normal Çıkış (Zero exit code) | `docker stop` Komutu | Daemon (Docker) Yeniden Başladığında |
| --- | --- | --- | --- | --- |
| **no** | N | N | N | N |
| **on-failure** | **Y** | N | N | **Y** |
| **always** | **Y** | **Y** | N* | **Y** |
| **unless-stopped** | **Y** | **Y** | N | N |

> **Not:**
> * **Non-zero exit code:** Bir hata oluştuğunu gösterir (Örn: Uygulama çöktü).
> * **Zero exit code:** Normal bir kapanışı gösterir (Örn: İş bitti ve `exit` dendi).
> 
> 

---

### 3. Uygulama: `always` Politikasını Test Etmek

Şimdi teoriyi pratiğe dökelim. `always` politikasını kullanarak bir konteyner başlatacağız, onu manuel olarak kapatacağız ve Docker'ın onu diriltmesini izleyeceğiz.

**Adım 1:** `neversaydie` (Asla Ölme) isminde, `always` politikasına sahip bir Alpine konteyneri başlatın.

```bash
$ docker run --name neversaydie -it --restart always alpine sh
/#

```

Terminaliniz otomatik olarak konteynerin içindeki `sh` kabuğuna bağlandı.

**Adım 2:** Konteyneri normal bir şekilde kapatmak için `exit` yazın.

```bash
/# exit

```

Bu komut "Zero exit code" (Normal çıkış) üretir. Normalde konteynerin kapanıp öylece kalması gerekirdi ama biz `--restart always` dediğimiz için Docker onu tekrar başlatmalıdır.

**Adım 3:** Durumu kontrol edin.

```bash
$ docker ps
CONTAINER ID   IMAGE    COMMAND   CREATED          STATUS          NAMES
1933623830bb   alpine   "sh"      35 seconds ago   Up 2 seconds    neversaydie

```

**Analiz:**

* **CREATED:** 35 saniye önce (İlk oluşturma zamanı).
* **STATUS:** **Up 2 seconds** (Sadece 2 saniyedir çalışıyor).
* **Sonuç:** Sen `exit` diyerek onu öldürdün, ama Docker onu hemen geri getirdi. Docker yeni bir konteyner yaratmadı, **aynı konteyneri** yeniden başlattı.

**Adım 4:** Yeniden başlatma sayısını kontrol edin (`inspect`).

```bash
$ docker inspect neversaydie | grep RestartCount
        "RestartCount": 1,

```

*(Windows PowerShell kullanıcıları `grep` yerine `Select-String -Pattern 'RestartCount'` kullanabilir.)*
Gördüğünüz gibi, Docker bu konteyneri 1 kez yeniden başlattığını kaydetmiş.

---

### 4. Kritik Fark: `always` vs `unless-stopped`

Metindeki en ilginç detaylardan biri bu iki politika arasındaki farktır.

**Senaryo:**

1. `--restart always` ile bir konteyner başlattın.
2. `docker stop` ile onu elle durdurdun.
3. Bilgisayarını (veya Docker servisini) yeniden başlattın.

**Sonuç:**

* **always:** Docker servisi açıldığı an, sen onu elle durdurmuş olsan bile, bu konteyneri tekrar başlatır. "Her zaman" kelimesini çok ciddiye alır.
* **unless-stopped:** Docker servisi açıldığında konteyneri başlatmaz. Çünkü sen onu en son elle durdurmuştun ("Durdurulmadığı sürece" mantığı).

Eğer konteynerin, sen onu kapattıktan sonra sistem yeniden başlasa bile kapalı kalmasını istiyorsan `unless-stopped` kullanmalısın.

---

### 5. Docker Compose / Stacks Kullanımı

İlerleyen bölümlerde göreceğimiz Docker Compose dosyalarında bu ayar şu şekilde yapılır:

```yaml
services:
  myservice:
    <Snip>
    restart_policy:
      condition: always | unless-stopped | on-failure

```

## Bölüm Sonu - Konteynır Komutları

* **`docker run`**: Yeni bir konteynır başlatmanın ana yoludur. İmaj adını verirsin ve Docker onu hayata geçirir.
* *Örnek:* `docker run -it ubuntu bash` (Ubuntu imajından interaktif bir Bash kabuğu başlatır).


* **`Ctrl-P-Q`**: Bağlı olduğun bir konteynırı **öldürmeden** terminalinden ayrılmanı (detach) sağlar. Günlük kullanımda en çok başvuracağın kısayoldur.
* **`docker ps`**: Sadece çalışan konteynırları listeler. Durmuş olanları (Exited) görmek için `-a` (all) bayrağını eklemelisin.
* **`docker exec`**: Çalışan bir konteynırın içine "ekstra" komutlar gönderir.
* *İnteraktif:* `docker exec -it <isim> bash` (İçeride yeni bir kabuk açar).
* *Uzaktan:* `docker exec <isim> ps` (İçeri girmeden komutu çalıştırıp sonucu getirir).


* **`docker stop`**: Konteynırı nazikçe durdurur. Önce `SIGTERM` sinyali gönderip 10 saniye bekler, eğer uygulama hala kapanmamışsa `SIGKILL` ile zorla sonlandırır.
* **`docker restart`**: Durmuş bir konteynırı eski ayarlarıyla tekrar ateşler.
* **`docker rm`**: Konteynırı sistemden tamamen siler. Eğer konteynır çalışıyorsa `-f` (force) ile önce durdurup sonra silebilirsin.

### İleri Seviye İzleme ve Hata Ayıklama

* **`docker inspect`**: Konteynırın IP adresinden başlangıç komutlarına kadar her türlü teknik detayını JSON formatında önünüze serer.
* **`docker debug`**: "Slim" (içi boş) imajlarda hayat kurtarır. Konteynırda shell olmasa bile dışarıdan bir alet çantasıyla içeri sızmanı sağlar (Pro/Business lisans gerektirir).

---
