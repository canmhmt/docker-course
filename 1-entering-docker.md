# Docker'a Giriş

Bu notun amacı Docker'ın ne yaptığını anlamak, temel kavramları öğrenmek ve ilk konteynerini çalıştırmaktır. Örnekler Linux konteynerleri içindir. Terminal kullanmayı bilmek yeterlidir; Ubuntu kurulumu için ayrıca `sudo` yetkisi gerekir.

## 1 - Docker nedir, neden kullanılır?

Docker, uygulamayı ve ihtiyaç duyduğu bağımlılıkları bir **imaj (image)** olarak paketleyip bu imajdan **konteyner (container)** çalıştırmayı sağlayan bir platformdur.

Örneğin bir Python uygulamasının belirli bir Python sürümüne ve kütüphanelere ihtiyacı olsun. Bunları imajın içine koyarak geliştirme ve sunucu ortamlarında aynı uygulama paketini kullanabilirsin. Ortama özel ayarlar, parolalar ve kalıcı veriler ayrıca yönetilir.

Docker şu konularda yardımcı olur:

- **Tutarlı ortam:** Uygulama ve bağımlılıklarını birlikte dağıtırsın.
- **Kaynak verimliliği:** Konteynerler genellikle ayrı işletim sistemi başlatan sanal makinelerden daha hafiftir. Yine de CPU, RAM ve disk tüketirler.
- **İzolasyon:** Uygulamaların süreçlerini, dosya sistemlerini ve ağ ortamlarını ayırabilirsin. İzolasyonun kapsamı yapılandırmaya bağlıdır.
- **Tekrarlanabilir dağıtım:** Aynı imajdan birden fazla konteyner oluşturabilirsin.
- **Sürüm yönetimi:** Önceki imajı saklayıp yeniden dağıtabilirsin. Bu, veritabanı değişikliklerini kendiliğinden geri almaz.

Konteyner teknolojileri Docker'dan önce de vardı. Docker, imaj oluşturma, paylaşma ve konteyner çalıştırma işlemlerini ortak bir araç setinde birleştirerek kullanımlarını kolaylaştırdı. Sanal makineler ve konteynerler birlikte de kullanılabilir; bir VM içinde Docker çalıştırmak yaygındır.

### Temel kavramlar

| Kavram | Anlamı |
| --- | --- |
| **Host** | Docker Engine'in çalıştığı fiziksel veya sanal makine. |
| **Image (imaj)** | Uygulama dosyaları, bağımlılıkları ve başlangıç ayarlarını içeren, salt okunur şablon. |
| **Container (konteyner)** | Bir imajdan oluşturulan, kendi çalışma durumu ve yazılabilir katmanı bulunan örnek. Çalışıyor veya durmuş olabilir. |
| **Dockerfile** | Bir imajın nasıl oluşturulacağını tarif eden metin dosyası. |
| **Registry** | İmajların saklandığı ve dağıtıldığı servis. Docker Hub bir örnektir. |
| **Volume** | Konteynerden bağımsız saklanan, Docker'ın yönettiği kalıcı veri alanı. |
| **Network** | Konteynerlerin ağ bağlantılarını düzenleyen yapı. |

Bir imajdan çok sayıda konteyner oluşturulabilir. Konteyner içinde dosya değiştirmek, kaynak imajı değiştirmez; değişiklik konteynerin yazılabilir katmanında kalır. İmajlar katmanlardan oluşur ve aynı katmanlar farklı imajlar arasında paylaşılabilir. [İmaj kavramı](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-an-image/)

Genel akış:

```text
Dockerfile + uygulama dosyaları
          |
       docker build
          |
         Image ---- docker push ----> Registry
          |                              |
       docker run                    docker pull
          |                              |
       Container <---- docker run ---- Image
```

### Docker ile sanal makine arasındaki fark

Sanal makine (VM), sanal donanım üzerinde kendi işletim sistemini ve kernel'ini çalıştırır. Linux konteynerleri ise çalıştıkları Linux ortamının kernel'ini paylaşır; her konteyner için ayrı kernel başlatılmaz. İmaj yine de kullanıcı alanına ait işletim sistemi araçlarını ve kütüphanelerini içerebilir.

| Özellik | Fiziksel makinede doğrudan uygulama | Sanal makine (VM) | Konteyner |
| --- | --- | --- | --- |
| Temel yapı | Donanım → işletim sistemi → uygulama | Donanım → hypervisor → konuk işletim sistemi → uygulama | Linux ortamı → konteyner çalışma altyapısı → uygulama |
| Kernel | Makinenin kernel'i | Her VM'in kendi kernel'i | Aynı Linux ortamındaki konteynerler kernel'i paylaşır |
| Ek kaynak ihtiyacı | Konteyner veya VM katmanı yoktur | Konuk işletim sistemi için de kaynak gerekir | Genellikle VM'den daha az ek kaynak gerekir |
| İzolasyon | İşletim sisteminin kullanıcı ve süreç sınırları | Ayrı kernel ve sanal donanım sınırı | Paylaşılan kernel üzerinde süreç izolasyonu |
| Uygun kullanım | Makineye doğrudan kurulan uygulamalar | Ayrı işletim sistemi veya kernel ihtiyacı | Uygulamaları bağımlılıklarıyla paketlemek |

Başlatma süresi uygulamaya ve ortama bağlıdır; fiziksel makinenin açılmasıyla bir konteyner sürecinin başlaması aynı işlem değildir. VM'ler genellikle daha güçlü bir izolasyon sınırı sunar; hiçbir model tek başına mutlak güvenlik sağlamaz.

**Taşınabilirliğin sınırı:** İmajın işletim sistemi ve işlemci mimarisi çalışma ortamıyla uyumlu olmalıdır. Örneğin `linux/amd64` ile `linux/arm64` farklı platformlardır; uygun imaj varyantı veya emülasyon gerekir. Windows ve macOS üzerinde Docker Desktop ile Linux konteynerleri çalıştırıldığında Linux kernel'i bir Linux sanallaştırma ortamından sağlanır. Windows'ta bunun için WSL 2 kullanılabilir. [Platform uyumluluğu](https://docs.docker.com/build/building/multi-platform/), [WSL 2 altyapısı](https://docs.docker.com/desktop/features/wsl/)

### Linux'ta izolasyon ve kaynak yönetimi

| Mekanizma | Görevi | Örnek |
| --- | --- | --- |
| **Namespaces** | Süreçlerin görebildiği sistem kaynaklarını ayırır. | Ayrı süreç listesi, ağ arayüzleri ve bağlama noktaları. |
| **Control groups (cgroups)** | Süreç gruplarının kaynak kullanımını izler ve sınırlar. | CPU kotası veya bellek sınırı. |

Docker varsayılan olarak her konteynere ayrı bir CPU/RAM üst sınırı koymaz. Gerekirse `docker run` sırasında `--cpus="1"` ve `--memory="256m"` gibi seçeneklerle sınır belirlersin. Bunlar kaynakları önceden ayırmaz; kullanım sınırlarını tanımlar. [Kaynak sınırları](https://docs.docker.com/engine/containers/resource_constraints/)

## 2 - Docker'ın bileşenleri

Docker Engine, istemci-sunucu mimarisiyle çalışır. Komut satırı istemcisi, API ve arka plandaki servis bu yapının parçalarıdır.

| Bileşen | Görevi |
| --- | --- |
| **Docker CLI (`docker`)** | Terminalde yazdığın komutları alır ve API isteğine dönüştürür. |
| **Docker API** | İstemci ile daemon arasındaki iletişim arayüzüdür. |
| **Docker daemon (`dockerd`)** | İmaj, konteyner, ağ ve volume gibi Docker nesnelerini yönetir. |
| **containerd** | Konteynerlerin yaşam döngüsünü yönetmek için Docker'ın kullandığı çalışma altyapısıdır. |
| **runc** | Linux'ta konteyner sürecini başlatmak için kullanılan düşük seviyeli OCI runtime örneğidir. |
| **Docker Desktop** | Engine ve CLI yanında Compose, grafik arayüz ve gerekli platform entegrasyonlarını sunan masaüstü uygulamasıdır. |

Varsayılan davranışla `docker run nginx:stable-alpine` yazıldığında:

1. CLI, isteği Docker API üzerinden daemon'a iletir.
2. Docker, imaj yerelde yoksa registry'den indirir.
3. İmajdan yeni bir konteyner oluşturur; dosya sistemi ve ağ ayarlarını hazırlar.
4. Çalıştırma altyapısı üzerinden imajın başlangıç komutunu çalıştırır.

CLI ile daemon aynı makinede olmak zorunda değildir. Bu notun uygulamaları yerel Docker ortamını varsayar. [Docker Engine](https://docs.docker.com/engine/), [Çalışma akışı](https://docs.docker.com/get-started/docker-overview/), [containerd](https://containerd.io/)

### Konteyner dünyasındaki standartlar

**OCI (Open Container Initiative)**, farklı araçların uyumlu çalışabilmesi için standartlar tanımlar:

- **Image Specification:** İmajın biçimi ve içeriği.
- **Runtime Specification:** Konteynerin çalıştırılmasına ilişkin kurallar.
- **Distribution Specification:** İmaj ve ilgili içeriğin registry üzerinden dağıtılması.

Docker konteyner dünyasındaki araçlardan biridir; konteyner kavramı yalnızca Docker'a ait değildir. [OCI standartları](https://opencontainers.org/about/overview/)

## 3 - Docker kurulumu

### Hangi sisteme ne kurulur?

| Ortam | Başlangıç seçeneği |
| --- | --- |
| Linux sunucu veya Linux geliştirme ortamı | Dağıtıma uygun Docker Engine kurulumu. |
| Windows geliştirme bilgisayarı | Desteklenen Windows sürümünde Docker Desktop; Linux konteynerleri için WSL 2 altyapısı. |
| macOS geliştirme bilgisayarı | İşlemci mimarisine uygun Docker Desktop. |
| Linux masaüstü | Docker Engine veya Docker Desktop for Linux. |

Docker Toolbox artık desteklenmiyor. Windows Home için Toolbox gerektiği bilgisi eskidir; desteklenen sistemlerde WSL 2 ile Linux konteynerleri kullanılabilir. Güncel gereksinimler için [Windows kurulumu](https://docs.docker.com/desktop/setup/install/windows-install/), [macOS kurulumu](https://docs.docker.com/desktop/setup/install/mac-install/) ve [Linux kurulumu](https://docs.docker.com/engine/install/) sayfalarına bak. [Toolbox'ın kullanım dışı bırakılması](https://github.com/docker-archive/toolbox)

Docker Desktop kurduysan uygulamayı başlat, Linux konteynerleri modunu kullan ve aşağıdaki Ubuntu kurulumunu atlayarak kurulum doğrulamasına geç. Desktop'ın WSL entegrasyonunu kullanırken aynı WSL dağıtımına ayrıca Engine kurman gerekmez.

### Ubuntu üzerinde Docker Engine

Bu adımlar **Docker'ın desteklediği Ubuntu sürümlerinde**, Bash terminalinde ve systemd kullanılan bir ortamda uygulanır. Diğer dağıtımlar için kendi kurulum sayfasını kullan. Aşağıdaki komutları sırayla çalıştır.

#### 1. Çakışan paketleri kaldır

Önceden dağıtım deposundan kurulan Docker veya runtime paketleri, Docker'ın resmî paketleriyle çakışabilir. Mevcut sunucuda paketlerin başka iş yükleri tarafından kullanılıp kullanılmadığını kontrol et; kaldırma işlemi onları etkileyebilir.

```bash
sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc docker-buildx podman-docker containerd runc | cut -f1)
```

Paketlerin bulunmadığı mesajı temiz bir kurulumda normaldir. Bu adım Docker verilerini silmek için kullanılmaz.

#### 2. Paket listesini ve gerekli araçları hazırla

```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
```

- `apt update` kullanılabilir paket listesini yeniler; kurulu paketleri yükseltmez.
- `ca-certificates` HTTPS sertifika doğrulaması için güvenilen sertifikaları sağlar.
- `curl` dosya indirmek için kullanılır.
- `install -m 0755 -d` anahtar dizinini uygun izinlerle oluşturur.

#### 3. Docker'ın imza anahtarını ekle

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

APT bu anahtarı depo imzasını doğrulamak için kullanır. `chmod a+r` anahtarın paket yöneticisi tarafından okunabilmesini sağlar.

#### 4. Resmî APT deposunu tanımla

```bash
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

`Suites` Ubuntu'nun kod adını, `Architectures` paket mimarisini, `Signed-By` ise bu depo için kullanılacak anahtarı belirtir. `EOF` kapanış satırını başında boşluk olmadan yaz.

#### 5. Docker paketlerini kur

```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Bu paketler sırasıyla Engine'i, CLI'ı, konteyner çalışma altyapısını, imaj oluşturma aracı Buildx'i ve Compose eklentisini kurar. Kurulum adımlarının güncel kaynağı: [Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/).

### Kurulumu doğrula

Ubuntu'da servisin durumuna bak:

```bash
sudo systemctl status docker --no-pager
```

Servis çalışmıyorsa başlat:

```bash
sudo systemctl start docker
```

Engine'e erişimi ve ilk konteyneri kontrol et:

```bash
sudo docker version
sudo docker info
docker compose version
sudo docker run --rm hello-world
```

- `docker version` çıktısında **Client** ve **Server** bölümlerini görmelisin.
- `docker info` çalışan Engine hakkında bilgi verir.
- `hello-world` indirilip çalışır, başarı mesajını yazar ve kapanır.
- `--rm` konteyner kapandığında onu otomatik siler; indirilen imaj kalır.

Docker Desktop'ta `systemctl` adımlarını atla ve `docker` komutlarını `sudo` olmadan çalıştır. Tek başına `docker --version` yalnızca CLI sürümünü gösterir; daemon'a bağlanabildiğini kanıtlamaz.

### İsteğe bağlı: Linux'ta sudo yazmadan Docker kullanmak

```bash
sudo groupadd -f docker
sudo usermod -aG docker "$USER"
```

Oturumu tamamen kapatıp yeniden aç veya mevcut terminalde yeni grup oturumu başlat:

```bash
newgrp docker
docker run --rm hello-world
```

**Bu işlem rootless kurulum değildir.** Docker daemon root olarak çalışmaya devam eder; `docker` grubu üyeliği root düzeyinde yetki sağlar. Gerçek rootless modda daemon da normal kullanıcı yetkileriyle çalışır. [Grup üyeliği](https://docs.docker.com/engine/install/linux-postinstall/), [Rootless mode](https://docs.docker.com/engine/security/rootless/)

Bundan sonraki örneklerde `docker` komutları `sudo` olmadan yazılmıştır. Linux'ta grup üyeliği veya rootless kurulum kullanmıyorsan komutların başına `sudo` ekle.


## 4 - İmaj adı, registry, repository ve tag

`nginx:stable-alpine` imaj adını parçalayalım:

| Parça | Bu örnekte | Anlamı |
| --- | --- | --- |
| Registry | `docker.io` | Alan adı belirtilmediği için Docker Hub kullanılır. |
| Repository | `library/nginx` | Docker Hub'daki resmî Nginx imajlarının bulunduğu depo. |
| Tag | `stable-alpine` | Kullanılacak imaj varyantını belirten etiket. |

Tam ad `docker.io/library/nginx:stable-alpine` şeklindedir. Genel adlandırma `registry/namespace/repository:tag` biçimini izler.

**Tag bir etikettir; içeriği sabitlemez.** Yayıncı aynı etiketi başka bir imaja yönlendirebilir. Etiket verilmezse varsayılan `latest` kullanılır; bu ad "en yeni sürüm" garantisi değildir. Belirli içeriği sabitlemek gerektiğinde `@sha256:...` biçimindeki digest kullanılır. [İmaj adları ve referansları](https://docs.docker.com/reference/cli/docker/image/pull/)

Temel komutlar:

```bash
docker pull nginx:stable-alpine
docker image ls
```

`pull` imajı indirir; konteyner başlatmaz. `docker run` ise varsayılan olarak imaj yerelde yoksa indirir ve yeni konteyner oluşturur. Registry'ye imaj göndermek için `docker push`, kimlik doğrulamak için `docker login` kullanılır.

Harbor, özel registry kurmak için kullanılabilen bir seçenektir; Docker'ı kendi bilgisayarına kurmak registry'yi kendiliğinden Harbor yapmaz. [Registry kavramı](https://docs.docker.com/get-started/docker-overview/#docker-registries)

## 5 - İlk uygulama: Nginx çalıştırmak

Bu örnekte bir web sunucusu başlatıp durumunu inceleyecek ve ardından kaldıracaksın. Docker'ın çalışıyor olması ve bilgisayarında 8080 portunun boş olması gerekir.

### Konteyneri başlat

```bash
docker run -d --name docker-giris-web -p 127.0.0.1:8080:80 nginx:stable-alpine
```

| Seçenek | Anlamı |
| --- | --- |
| `-d` | Konteyneri arka planda çalıştırır. |
| `--name docker-giris-web` | Konteynere kullanılabilir bir ad verir. |
| `-p 127.0.0.1:8080:80` | Host'un yerel 8080 portunu konteynerin 80 portuna yönlendirir. |
| `nginx:stable-alpine` | Kullanılacak imajdır. |

Aynı bilgisayarda tarayıcıdan [http://localhost:8080](http://localhost:8080) adresini aç. Nginx'in karşılama sayfasını görmelisin.

`-p` sıralaması `HOST_IP:HOST_PORT:CONTAINER_PORT` şeklindedir. Burada port yalnızca yerel arayüze bağlanır. `-p 8080:80` yazılsaydı varsayılan olarak host'un tüm ağ arayüzlerinde yayınlanırdı. [Port yayınlama](https://docs.docker.com/engine/network/port-publishing/)

### Durumunu ve loglarını incele

```bash
docker ps
docker logs docker-giris-web
docker inspect docker-giris-web
docker exec docker-giris-web cat /etc/os-release
```

- `ps` çalışan konteynerleri, `ps -a` durmuş olanları da listeler.
- `logs` uygulamanın standart çıktısını ve hata çıktısını gösterir; tarayıcı isteğinden sonra erişim kaydı görebilirsin.
- `inspect` yapılandırma, ağ ve durum bilgisini JSON olarak verir.
- `exec` çalışan konteynerde ek bir komut başlatır. Buradaki `cat` konteyner içindeki dosyayı okur.

Etkileşimli kabuk için `docker exec -it docker-giris-web sh` kullanılabilir. `-i` standart girdiyi açık tutar, `-t` terminal ayırır. Bu kabuktan `exit` ile çıkmak Nginx'in ana sürecini durdurmaz.

### Durdur, yeniden başlat ve sil

```bash
docker stop docker-giris-web
docker ps -a
docker start docker-giris-web
```

`stop` konteyneri durdurur; konteyner ve yazılabilir katmanı kalır. `start` aynı konteyneri tekrar başlatır. `run` ise her çağrıldığında yeni konteyner oluşturur.

Deneme bittiğinde:

```bash
docker stop docker-giris-web
docker rm docker-giris-web
```

`rm` konteyneri ve yazılabilir katmanını siler; imajı silmez. Konteyner, ana süreci çalıştığı sürece çalışır. Bu yüzden `hello-world` hemen kapanırken Nginx istek beklemeye devam eder. [Konteyner yaşam döngüsü](https://docs.docker.com/engine/containers/run/)

### Ağ hakkında iki temel bilgi

Konteyner içindeki `localhost` normalde o konteynerin kendisidir; host bilgisayarı veya başka bir konteyner değildir. Aynı kullanıcı tanımlı bridge ağına bağlanan konteynerler birbirlerine adlarıyla erişebilir.

Aynı ağda bulunmak dosya sistemlerini paylaşmak anlamına gelmez. Konteynerler arası ağ iletişimi ile host üzerinden port yayınlamak ayrı konulardır. [Docker ağları](https://docs.docker.com/engine/network/)

## 6 - Veriler nerede kalır?

Konteynerin kendi yazılabilir katmanındaki dosyalar durdurma ve yeniden başlatma sırasında korunur, **konteyner silindiğinde kaybolur**. Kalıcı veri için volume veya bind mount kullanılır.

| Yöntem | Kullanım |
| --- | --- |
| **Volume** | Docker'ın yönettiği veri alanıdır. Konteyner silinse de adlandırılmış volume kalır. |
| **Bind mount** | Host'taki belirli bir dosya veya klasörü konteynere bağlar. Yerel kaynak koduyla çalışırken kullanışlıdır. |

Aşağıdaki deneyde ilk konteyner bir dosya yazıp silinir; ikinci konteyner aynı volume'dan dosyayı okur:

```bash
docker volume create docker-giris-veri
docker run --rm --mount type=volume,source=docker-giris-veri,target=/veri alpine:3 sh -c 'echo Merhaba > /veri/not.txt'
docker run --rm --mount type=volume,source=docker-giris-veri,target=/veri alpine:3 cat /veri/not.txt
```

Çıktıda `Merhaba` görmelisin. `source` volume adıdır; `target` konteyner içindeki bağlama yoludur. `--rm` konteynerleri temizler, bu adlandırılmış volume'u silmez.

Deneme verisine artık ihtiyacın yoksa:

```bash
docker volume rm docker-giris-veri
```

Bu son komut volume'u ve içindeki deneme dosyasını siler. Volume kullanmak yedek almak değildir; önemli veriler ayrıca yedeklenmelidir. [Veri kalıcılığı](https://docs.docker.com/get-started/docker-concepts/running-containers/persisting-container-data/)

## 7 - Kendi imajına ilk adım: Dockerfile

Hazır imaj kullanmanın yanında kendi uygulama dosyalarını içeren imaj da oluşturabilirsin. Boş bir çalışma klasöründe aşağıdaki iki dosyayı oluştur.

`index.html`:

```html
<!doctype html>
<html lang="tr">
  <head><meta charset="utf-8"><title>Docker'a Giriş</title></head>
  <body><h1>Merhaba Docker</h1></body>
</html>
```

`Dockerfile` (dosya uzantısı yok):

```dockerfile
FROM nginx:stable-alpine
COPY index.html /usr/share/nginx/html/index.html
```

- `FROM` temel imajı seçer.
- `COPY` sayfanı imajın içine ekler.
- Başlangıç komutu Nginx temel imajından devralındığı için ayrıca `CMD` yazmak gerekmez.
- İmajın içine alınan dosya bir kopyadır; yerel HTML dosyasını değiştirirsen imajı yeniden oluşturmalısın.

Bu klasörde çalıştır:

```bash
docker build -t docker-giris-site:1.0 .
docker run -d --name docker-giris-site -p 127.0.0.1:8080:80 docker-giris-site:1.0
```

`-t` imaja ad ve etiket verir. Sondaki `.` mevcut klasörü **build context**, yani derleme girdilerinin bulunduğu yer olarak seçer. Tarayıcıdan [http://localhost:8080](http://localhost:8080) adresini açtığında bu kez kendi sayfan görünür.

Daha büyük projelerde `.dockerignore` ile gereksiz dosyaları ve gizli bilgileri build context dışında tutarsın. Dockerfile'daki `EXPOSE` da tek başına host portu yayınlamaz; bunun için çalıştırırken `-p` gerekir. [Dockerfile yazmak](https://docs.docker.com/get-started/docker-concepts/building-images/writing-a-dockerfile/), [EXPOSE](https://docs.docker.com/reference/dockerfile/#expose)

Örneği temizle:

```bash
docker stop docker-giris-site
docker rm docker-giris-site
```

### Docker Compose ne zaman devreye girer?

Uygulaman web sunucusu, API ve veritabanı gibi birden fazla servisten oluştuğunda komutları tek tek yazmak zorlaşır. **Docker Compose**, servisleri, ağları ve volume'ları `compose.yaml` dosyasında tanımlayıp birlikte yönetmeyi sağlar. Tek servis için de kullanılabilir.

Bir Compose projesinde `docker compose up -d` servisleri oluşturup başlatır, `docker compose logs` logları gösterir, `docker compose down` projenin konteynerlerini ve ağlarını kaldırır. `down` varsayılan olarak adlandırılmış volume'ları silmez; `-v` eklendiğinde veriler de silinebilir. Bu komutlar için önce bir Compose dosyası gerekir. [Compose'a giriş](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-docker-compose/)

## 8 - Kurulum sonrası kısa notlar

### İsteğe bağlı: Linux Engine'de log boyutunu sınırlamak

`json-file` sürücüsünde log rotasyonu ayarlamak için `/etc/docker/daemon.json` dosyasında aşağıdaki alanlar kullanılabilir. Dosya zaten varsa mevcut ayarları koruyarak aynı JSON nesnesine ekle:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

`max-size` dosya başına boyut eşiğini, `max-file` tutulacak dosya sayısını belirler. Değerler tırnak içinde yazılır. Bu ayarlar Docker'ın topladığı standart çıktı/hata logları içindir; uygulamanın kendi dosyalarına yazdığı logları sınırlamaz.

Ayarı uygulamak için:

```bash
sudo systemctl restart docker
```

Servisi yeniden başlatmak çalışan konteynerleri etkileyebilir. Yeni varsayılanlar **yeni oluşturulan konteynerlere** uygulanır; mevcut konteynerler eski log ayarlarını korur. Docker Desktop'ta aynı ayarlar uygulamanın **Settings → Docker Engine** bölümünden yönetilir. [Log yapılandırması](https://docs.docker.com/engine/logging/drivers/json-file/)

### Sık karşılaşılan durumlar

| Belirti | İlk kontrol |
| --- | --- |
| `Cannot connect to the Docker daemon` | Engine servisi veya Docker Desktop çalışıyor mu? `docker context ls` doğru ortamı gösteriyor mu? |
| Docker socket için `permission denied` | Linux'ta `sudo` ile dene veya grup üyeliğinin yeni oturuma yansıdığını kontrol et. |
| `port is already allocated` | Host portu dolu olabilir; örneğin `127.0.0.1:8081:80` kullanıp tarayıcıda 8081'i aç. |
| Konteyner adı zaten kullanımda | `docker ps -a` ile mevcut konteyneri bul; onu başlat veya yeni bir ad seç. |
| Konteyner hemen kapanıyor | `docker ps -a` ve `docker logs KONTEYNER_ADI` ile bak; ana sürecin işi bitmiş veya hata vermiş olabilir. |
| `no matching manifest` | İmajın işletim sistemi/mimari desteğini ve Linux konteynerleri modunda olduğunu kontrol et. |

Bu çalışmadan sonra bir imajı indirip konteyner başlatabilir, port yayınlayabilir, log okuyabilir, konteyneri durdurup silebilir ve veriyi bir volume'da koruyabilirsin. Sonraki adımlar Dockerfile komutları, Compose ile çok servisli uygulamalar ve kullanıcı tanımlı ağlardır.
