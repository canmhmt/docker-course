## Docker'a girmeden kısa tarihçe
1. Eskiden her iş uygulaması için ayrı, genellikle gereğinden güçlü bir fiziksel sunucu gerekiyordu.

2. VMware ile bu durum değişti ve birden fazla uygulama aynı sunucuda çalışabilir hale geldi.

3. Ardından gelen konteynerler, daha verimli ve taşınabilir bir sanallaştırma çözümü sundu.

4. Başta karmaşık olan konteyner teknolojisini Docker herkes için erişilebilir hale getirdi.

5. Wasm (WebAssembly) ve Yapay Zekâ (AI) gibi yeni teknolojiler inovasyonu yönlendiriyor.

6. Docker ekosistemi, bu yeni teknolojilerle birlikte evrim geçiriyor.


# 1 - Docker'a Giriş 
- Docker, uygulamaları hafif, taşınabilir ve izole (container) şekilde çalıştırmamızı sağlayan bir platformdur. Yani bir uygulamanın bağımlılıklarını ve uygulamayı bir konteyner'e koyup istediğimiz makinede sorunsuz çalıştırmamızı sağlar.

## Docker Neden Kullanılır?
### Taşınabilirlik
- Kod kendi ortamıyla birlikte gelir. Bir yerde çalışıyorsa, başka her makinede de aynı şekilde çalışır.

### Hafiflik
- Container'lar VM gibi değildir, çok daha hafif ve hızlıdır. Aynı makinede onlarca container çalıştırılabilir.

### Kolay Dağıtım
- Uygulamaları paketleyip farklı sunuculara aktarmak çok kolaydır.

### İzolasyon
- Her container kendi iç dünyasını görebilir, aynı makinede olan container'lar birbirlerini göremez (aynı network üzerinde değillerse).

### Versiyon Kontrolü
- Uygulamanın her sürümünü "image" olarak saklayarak istediğimiz zaman istediğimiz sürüme dönebiliriz.

## Docker vs VM
### Docker ile VM arasındaki farklar:
#### 1 - İşletim Sistemi ve Kernel (En önemli fark)
- VM kendi işletim sistemini ve kernel'ini yükler, ayrı bir katman oluşturur (Hypervisor). Bu da ekstra bir katman oluşturulmasından dolayı host sisteme ekstra yük bindirir.
- Fakat konteyner (Docker) host işletim sisteminin kernel'ını kullandığından dolayı çok daha hafif yapıdadır, sisteme ekstra yük bindirmez.

#### 2 - Taşınabilirlik ve Uyumluluk
- Docker her sistemde çalışabildiği için taşınabilirlik ve uyumluluk açısından çok esnektir fakat VM farklı donanımlarda veya host sistemlerde uyumsuzluk gibi sorunlar çıkarabilir.

#### 3 - İzolasyon
- Docker izolasyon sağlar fakat izolasyon konusunda VM tam izolasyon sağladığından daha güvenlidir.

| Özellik                | 🖥️ **Bare Metal**                                        | 💾 **Sanal Makine (VM)**                        | 📦 **Docker (Container)**                           |
| ---------------------- | --------------------------------------------------------- | ----------------------------------------------- | --------------------------------------------------- |
| **Tanım**              | Fiziksel makine üzerine doğrudan kurulan işletim sistemi  | Donanım üzerinde sanallaştırılmış tam sistem    | İşletim sistemi üzerinde çalışan hafif container    |
| **Katman Yapısı**      | Donanım → OS → Uygulama                                   | Donanım → Hypervisor → OS (guest) → Uygulama    | Donanım → OS → Docker Engine → Container            |
| **İşletim Sistemi**    | Tek bir işletim sistemi                                   | Her VM kendi işletim sistemiyle gelir           | Host OS kernel’i paylaşılır, ek işletim sistemi yok |
| **Kaynak Kullanımı**   | En verimli (donanıma doğrudan erişim)                     | Ağır (her VM ayrı OS taşır)                     | Hafif (sadece uygulama + bağımlılıklar)             |
| **Başlatma Süresi**    | En hızlı (fiziksel açılış süresi)                         | Yavaş (dakikalar)                               | Çok hızlı (saniyeler)                               |
| **İzolasyon**          | Fiziksel ayrım, yüksek güvenlik                           | Güçlü (tam sistem yalıtımı)                     | Orta düzey (process-level isolation)                |
| **Taşınabilirlik**     | Zayıf (makineye bağlı)                                    | Orta (VM image taşınabilir)                     | Yüksek (Docker image her yerde aynı çalışır)        |
| **Bakım ve Yönetim**   | Zor (donanım arızasında müdahale gerekebilir)             | Orta (snapshot, backup alınabilir)              | Kolay (script’lerle hızlı yönetim)                  |
| **Kullanım Senaryosu** | Donanım gerektiren özel işler (veritabanı, GPU işlemleri) | Farklı OS çalıştırmak, güçlü izolasyon ihtiyacı | Mikroservis, CI/CD, modern uygulama dağıtımı        |

## Namespaces ve Control Groups (cgroups)
- Docker konteynerlarının kalbinde bu iki Linux özelliği bulunmaktadır. Bu özellikler olmasa Docker diye bir şey olamazdı.
- Bu ikisi birlikte çalışınca, namespaces izolasyonu sağlarken (container izolasyonu), cgroups da kaynak kullanımını yöneterek (hangi container ne kadar kaynak kullanabilir) "hafif bir sanallaştırma" sağlanır. İşte konteyner yapısı budur.

| Özellik   | Namespaces                                         | Control Groups (cgroups)                     |
| --------- | -------------------------------------------------- | -------------------------------------------- |
| Ne yapar? | İşlemleri ve kaynakları **izole eder**             | İşlemlerin **kaynak kullanımını sınırlar**   |
| Amaç      | İzolasyon (görünürlüğü sınırlandırmak)             | Kaynak yönetimi ve sınırlandırma             |
| Örnek     | Her container kendi ağını ve dosya sistemini görür | Her container RAM ve CPU sınırına sahip olur |

## Docker Platformu Temel Bileşenleri
- Docker Platformunun iki temel bileşeni vardır, Docker Engine ve Docker CLI.
### 1. Docker Engine
- Docker Engine Docker'ın kalbidir. Konteynerları çalıştıran asıl sistemdir. Üç ana yapıdan oluşur:
  **Docker Daemon**  
  **Docker Rest API**  
  **Docker CLI**  
- Bunlar birlikte çalışarak konteynerları, image'ları, networkleri ve volume'ları yönetir.

#### Docker Daemon (dockerd)
- Asıl işi yapan motor budur. Container'ları başlatır, durdurur, image indirir, volume ve network kurar.
- Kullanıcı doğrudan daemon'a dokunmaz, CLI veya Rest API üzerinden erişir.

#### Docker Rest API
- Daemon ile konuşmanın standart arayüzüdür. Docker CLI ile Docker Daemon arasında iletişimi sağlar.

### 2. Docker CLI
- Kullanıcının Docker ile iletişim kurduğu komut satırı arayüzü. Örneğin `docker run` komutu buradan gelir.

### Kısaca nasıl çalışır?
    Sen terminalden docker run komutunu yazarsın.
    Docker CLI bu isteği Rest API aracılığıyla Docker Daemon’a iletir.
    Docker Daemon, image’a bakar, gerekiyorsa indirir.
    Image’dan container yaratır ve çalıştırır.
    Container çalışırken sen onun durumunu izleyebilir, durdurabilir ya da silebilirsin.

| Katman        | Görev                                        |
| ------------- | -------------------------------------------- |
| Docker CLI    | Kullanıcının komutunu alır                   |
| REST API      | CLI ile Daemon arasındaki haberleşme yoludur |
| Docker Daemon | Asıl işlemleri yapar (container, image vs.)  |

Not: Docker sürümleri genellikle Docker CE (Community Edition) ve Docker EE (Enterprise Edition) olarak ikiye ayrılır. CE ücretsiz ve açık kaynaklı sürüm iken EE ücretli sürümüdür.

## Konteyner Dünyasındaki Önemli Standartlar

### OCI (Open Container Initiative)
- Standartları belirleyen kurul.

#### OCI'nin 3 standardı vardır
1. image-spec -> Docker imaj dosyalarının nasıl olacağını belirler. Docker'da **BuildKit**
2. runtime-spec -> Konteyner nasıl çalıştırılır? Docker'da **runc**
3. distribution-spec -> İmajlar nasıl dağıtılır? Docker'da **Docker Hub**

# 2 - Docker Kurulumu
## Docker Toolbox
- Windows 10 Home ve altı sürümler Docker desteklemez, bu sürümler için Docker Toolbox indirilmesi gerekir.
## Hangi sisteme hangi Docker kurulacak?
- Linux --> Docker Engine CE
- Mac OsX --> Docker Desktop for Mac
- Windows 10 Pro ve üstü --> Docker for Windows
- Windows 7-8 ve 10 Home --> Docker Toolbox

## Adım Adım Kurulum

### *0. Önceki kalan Docker paketleri temizlenir.*

### 1. Update ve Upgrade
- Sistem paketleri güncellenir.

### 2. ca-certificates curl gnupg lsb-release kurulumları
```bash
sudo apt install ca-certificates curl gnupg lsb-release
```
- Burada;
**ca-certificates** - HTTPS baplantılarının güvenilir olmasını sağlayan sertifika otoritelerini içerir.
**curl** - İnternet üzerinden veri indirmeyi sağlar (GPG (GNU Privacy Guard Key) çekerken kullanacağız)
**gnupg** - GPG anahtarlarını yönlendirmek için gerekli araç (imza doğrulama)
**lsb-release** - Linux dağıtımı ve sürümü hakkında bilgi almak için kullanılır (depo URL sinde kullanılacak)

### 3. Docker GPG anahtarını ekleme
```bash 
sudo mkdir -p /etc/apt/keyrings
```

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```
- Burada;
- Curl ile depo anahtarını çektik pipe kullanarak ASCII formatındaki gpg anahtarını binary formatına çevirip docker.gpg olarak /etc/apt/keyrings klasörüne kaydettik. (Apt paket yöneticisinin gpg anahtarlarını sakladığı dizin.)

### 4. Resmi Docker Deposunu Eklemek
```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) stable" \ 
 | sudo tee /etc/apt/sorces.list.d/docker.list > /dev/null
```

- Burada;
- **echo "deb []"** komutu ile Yeni bir depo tanımı oluşturuluyor.
- **arch=$(dpkg --print-architecture)** işlemci mimarisini otomatik algılar.
- **signed-by=/etc/apt/keyrings/docker.gpg** İmlalanmıs gpg dosyasının konumunu belirler.
- **lsb_release -cs** Mevcut ubuntu sürüm kod adını (jammy, focal) otomatik ekler. stable kararlı sürüm olduğunu belirtir.
- **| sudo tee /etc/apt/sources.list.d/docker.list** çıktıyı docker repo dosyasına yazar.
- **> /dev/null** - Ekrana print basmaması için çıktı buraya yönlendirilir. (tee komutu sudo ile dosyaya veri yazmamızı sağlar ve aynı anda print eder.)

### 5. Paket listesini güncelle ve Docker'ı kur

```bash
sudo apt update
```

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

- Burada;
- **docker-ce** docker community edition (ce) ana motor.
- **docker-ce-cli** docker komut satırı aracı.
- **containerd.io** docker'ın container çalıştırmak için kullandığı runtime (bağımlılık, altyapı)
- **docker-buildx-plugin** gelişmiş image inşa aracı
- **docker-compose-plugin** docker compose v2

#### Kurulum Sonrası Yapılacaklar

- Kurulum doğru mu kontrol et.
```bash
docker --version
```

```bash
docker compose version
```

```bash
sudo systemctl status docker
```

- **Rootsuz kullanım ayarı** -> 
```bash
sudo usermod -aG docker $USER
newgrp docker
```

- **log boyutu ayarlama**
```bash
/etc/docker/daemon.json
``` 
- Bu dosya üzerinden container içerisindeki logların boyutlarını ayarlayabiliriz.

- Not: **docker doctor** komutu ile Docker'ın sağlığı kontrol edilebilir.

# 3 - Image Registry
- Container Registry, Docker Registry olarak da bilinir.
- Docker imajlarını depolayıp dağıtabildiğimiz bir altyapı sunan servislere verilen addır.
- En bilineni Docker Engine'nin varsayılan olarak baktığı Docker Hub'dır. (Local ortamda kurulduysa Harbor.)


