# Docker Engine

## Docker Engine Giriş

- Bu bölümün temel amacı, Docker'ı sadece 'kullanmak' ile master seviyesinde anlamak arasındaki farkı kapatmaktır.

## Bölümün Odak Noktası ve Kapsamı

- Docker'ı bu detayları bilmeden de kullanabilirsiniz fakat gerçekten uzmanlaşmak istiyorsanız alt tarafta neler olup bittiğini bilmek zorundasınız.

#### Docker Engine Genel Alt Başlıklar

- **Docker Engine - The TLDR**
- **The Docker Engine Genel Yapı**
- **OCI Etkisi**
- **runc & containerd**
- **shim**

## Docker Engine TLDR (Kısa Özet)

- "Docker Engine" ifadesi aslında Doker konteynerlarını yöneten ve çalıştıran server-side bileşenlerin tamamını kapsayan bir jargondur. 

- **Docker Engine** tek bir araç gibi görünür fakat arka planda high-level-runtime, low-level-runtime, API, image builder ve shims gibi parçaların (companent) koordineli çalışmasıdır.

### Docker Engine Companentleri

```plaintext
┌────────────┐       ┌─────────────────────────────────────────┐
│            │       │              Docker Engine              │
│ $ docker   │       │  ┌───────┐      ┌──────────┐            │
│    CLI     ├──────>│  │ {API}  │ ───> │  daemon  │            │
│            │       │  └───────┘      └────┬───▲──┘            │    ┌─────────┐
└────────────┘       │                      │   │               │    │ Plugins │
                     │                 ┌────▼───┴────┐          ├────┤    &    │
                     │                 │ containerd  │          │    │  other  │
                     │                 └────┬───▲────┘          │    └─────────┘
                     │                      │   │               │
                     │                 ┌────▼───┴────┐          │
                     │                 │    runc     │          │
                     │                 └────┬────────┘          │
                     └──────────────────────┼───────────────────┘
                                            │
                                            ▼
                                ┌──────────────────────┐
                                │ Running container(s) │
                                └──────────────────────┘
```


## Docker Engine'in Evrimi: Monolitik Yapıdan Parçalara

- **runc** ve **containerd** isimleri, resmi dokümantasyonda olduğu gibi her zaman küçük harfle yazılacaktır (cümle başında bile olsalar).

### 1. Yolun Başında: İki Büyük Dev

Docker ilk çıktığında sadece iki ana bileşenden oluşuyordu:

* **Docker daemon:** Her şeyi yapan devasa (**monolithic**) bir yapıydı. API, imaj oluşturma, network ve volume yönetimi gibi tüm kodlar bu tek dosyanın içindeydi.
* **LXC (LinuX Containers):** Linux çekirdeği (kernel) ile konuşan, konteynırın temelleri olan *namespaces* ve *cgroups* yapılarını kuran "asıl işçi" buydu.

### 2. LXC'den Kurtuluş ve libcontainer

Docker zamanla LXC'yi kullanmayı bıraktı. Bunun iki sebebi vardı:

* **Platform Bağımlılığı:** LXC sadece Linux'a özeldi, Docker ise çok platformlu (Windows vb.) olmak istiyordu.
* **Kontrol Kaybı:** LXC Docker'dan bağımsız gelişiyordu ve Docker'ın hızına yetişemiyordu.

Sonuç olarak Docker, kendi aracı olan **libcontainer**'ı geliştirdi. Bu araç, işletim sisteminden bağımsız (platform-agnostic) bir şekilde kernel seviyesindeki konteynır bloklarına erişim sağladı. Sonradan **libcontainer** geliştirilerek **runc** haline getirildi.

### 3. Devasa (Monolithic) Yapının Parçalanması

Docker daemon'ın her işi tek başına yapması zamanla şu sorunlara yol açtı:

* **Yavaşlama:** Sistem hantallaştı.
* **Ekosistem Beklentisi:** Kullanıcılar her şeyi içeren tek bir blok yerine daha esnek yapılar istiyordu.
* **İnovasyon Zorluğu:** Tek parça dev yazılımları geliştirmek ve yenilemek zordur.

Bu yüzden Docker, "parçala ve yönet" stratejisine geçti. Her özelliği küçük, uzmanlaşmış araçlara böldü:

* **runc:** Alt seviye çalışma zamanı (low-level runtime) görevini üstlendi.
* **containerd:** Üst seviye çalışma zamanı (high-level runtime) görevini üstlendi.
* **İmaj Yönetimi:** Hatta en son (Docker Desktop 4.27.0 ile), imaj yönetimi bile daemon'dan alınıp containerd'ye devredildi.

### 4. Sonuç ve Bugünkü Durum

Bugün bu parçalar (runc ve containerd) sadece Docker tarafından değil; **Kubernetes**, **Firecracker** ve **Fargate** gibi dev projeler tarafından da ortaklaşa kullanılıyor. Docker artık her işi kendi yapan bir dev değil, uzman araçları koordine eden bir şef gibidir.

---

**Özetle:** Docker, "her şeyi ben yaparım" diyen hantal bir yapıdan; işi uzmanlarına (**runc**, **containerd**, **libcontainer**) dağıtan çevik ve modüler bir yapıya dönüştü.


## OCI (Open Container Initiative) Etkisi

Docker, Inc. motoru parçalarına ayırırken (refactoring), aynı zamanda **OCI** adı verilen bir oluşum konteynır dünyası için evrensel standartlar belirlemeye çalışıyordu. Bu standartlar, farklı araçların birbiriyle uyumlu çalışmasını sağlar.

### 1. Üç Temel Standart (Specifications)

* **Image Specification (image-spec):** Bir konteynır imajının yapısının nasıl olması gerektiğini belirler.
* **Runtime Specification (runtime-spec):** Bir konteynırın nasıl çalıştırılacağını ve yönetileceğini belirler.
* **Distribution Specification (distribution-spec):** İmajların *registry* (kayıt dizini) üzerinden nasıl dağıtılacağını düzenler.

### 2. Docker ve OCI İlişkisi

Docker, Inc. sadece bu standartlara uymakla kalmamış, aynı zamanda OCI'nın kurucu üyelerinden biri olmuştur.

* Docker hala bu standartlara kod katkısı sağlamaya ve geleceğini şekillendirmeye devam etmektedir.
* 2016'dan beri çıkan tüm Docker sürümleri bu OCI standartlarını eksiksiz uygular.

### 3. Docker İçindeki OCI Uygulamaları

* **runc:** OCI *runtime-spec* (çalışma zamanı standardı) için geliştirilmiş **referans** uygulamadır; Docker bunu konteynır yaratmak için kullanır.
* **BuildKit:** OCI *image-spec* (imaj standardı) ile uyumlu imajlar oluşturmak için Docker'ın kullandığı araçtır.
* **Docker Hub:** OCI *distribution-spec* (dağıtım standardı) ile uyumlu bir merkezi kayıt dizinidir (*registry*).

---

**Özetle:** OCI, konteynır dünyasının "anayasasını" yazar. Docker da bu anayasaya göre hareket ederek imajlarını hazırlar (**BuildKit**), dağıtır (**Docker Hub**) ve çalıştırır (**runc**).


## runc: Konteynırların Alt Seviye İşçisi

### 1. OCI Referans Uygulaması

* **runc**, OCI tarafından belirlenen *runtime-spec* (çalışma zamanı standardı) için oluşturulmuş **referans uygulama**dır.

### 2. Teknik Yapısı ve Görevi

* **runc**, aslında **libcontainer** kütüphanesinin üzerine inşa edilmiş, hafif (lightweight) bir **CLI** (komut satırı) aracıdır.
* Temel görevi; işletim sistemi çekirdeğiyle (kernel) iletişime geçerek konteynırı gerçekten inşa etmek veya silmek gibi yaşam döngüsü olaylarını fiziksel olarak gerçekleştirmektir.
* Bu nedenle kendisine **low-level runtime** (alt seviye çalışma zamanı) denir ve OCI katmanında faaliyet gösterdiği söylenir.

### 3. Kullanım Deneyimi ve Farkı

* **runc** tek başına indirilip kullanılabilir, ancak Docker Engine'in sunduğu zengin özelliklerin (network, storage yönetimi vb.) neredeyse hiçbirine sahip değildir.
* Docker, **runc**'ı kendi bünyesinde bir alt bileşen olarak kullanır. Böylece kullanıcılar, arka planda OCI uyumlu standart bir konteynır yapısına sahip olurken, ön planda Docker'ın kullanıcı dostu deneyiminden faydalanırlar.

### 4. containerd ile İş Birliği

Hem Docker hem de Kubernetes, **runc**'ı varsayılan olarak şu ikili yapıda kullanır:

* **containerd (high-level runtime):** Konteynırın yaşam döngüsü olaylarını (başlatma, durdurma isteği vb.) yöneten üst düzey yöneticidir.
* **runc (low-level runtime):** Bu emirleri alıp, kernel ile konuşarak işin "ameleliğini" yapan, konteynırı fiilen kuran veya kaldıran alt düzey işçidir.

---

**Özetle:** **runc**, Docker dünyasının "atom parçalayıcısıdır". Çok düşük seviyededir, sadece konteynır yaratmayı ve silmeyi bilir ama bu işi tam standartlara (OCI) uygun yapar.


## containerd: Üst Seviye Yönetici

### 1. High-Level Runtime (Üst Seviye Çalışma Zamanı)

* **containerd**, konteynırların başlatılması, durdurulması ve silinmesi gibi **lifecycle events** (yaşam döngüsü olaylarını) yönetir.
* Ancak bu işleri fiziksel olarak kendi yapmaz; gerçek işi yapması için bir **low-level runtime**'a ihtiyaç duyar.
* Çoğu zaman **runc** ile eşleşir; ancak **shim** katmanı sayesinde **runc** yerine farklı alt seviye araçlarla (örneğin WebAssembly için farklı runtime'lar) çalışabilecek esnekliğe sahiptir.

### 2. Kapsamın Genişlemesi ve Modülerlik

* Başlangıçta sadece yaşam döngüsü yöneten küçük bir araç olması planlanmıştı; ancak zamanla imaj yönetimi (**image management**), ağ (**network**) ve depolama (**volumes**) yetenekleri de eklendi.
* Bu genişlemenin ana sebeplerinden biri **Kubernetes** gibi projelerin beklentileridir; çünkü Kubernetes, imajların çekilmesi (pull) ve itilmesi (push) gibi işlemleri de **containerd**'nin yapmasını ister.
* En önemli özelliği **modüler** olmasıdır; yani Kubernetes gibi bir proje **containerd**'yi kullandığında, sadece ihtiyacı olan parçaları alıp geri kalanını bırakabilir.

### 3. Docker'dan CNCF'e Yolculuk

* **containerd**, başlangıçta Docker, Inc. tarafından geliştirilmiş ancak sonradan **CNCF** (Cloud Native Computing Foundation) vakfına bağışlanmıştır.
* Şu anda "graduated" (mezun) seviyesinde bir CNCF projesidir; bu da onun tamamen stabil, güvenilir ve **production-ready** (üretime hazır) olduğu anlamına gelir.

---

**Özetle:** **containerd**, operasyonun beynidir. Docker daemon'dan ayrılan bu parça, bugün tek başına hem Docker'ın hem de Kubernetes'in omurgasını oluşturur. 


## Yeni Bir Konteynır Başlatma: Adım Adım Akış
`$ docker run -d --name ctr1 nginx`

### 1. İstek ve API Katmanı

* Süreç, senin terminale (Docker CLI) yazdığın komutla başlar.
* Docker client, bu komutu bir **API request**'e (API isteği) dönüştürür ve **daemon**'a gönderir.
* Bu iletişim kanalı işletim sistemine göre değişir: Linux'ta bir **local socket** olan `/var/run/docker.sock` kullanılırken, Windows'ta `\pipe\docker_engine` üzerinden gerçekleşir.

### 2. Daemon'dan containerd'ye Geçiş

* **daemon** isteği alır, bunun "yeni bir konteynır yaratma" isteği olduğunu anlar ve topu **containerd**'ye atar.
* **Önemli Not:** Yazar burada tekrar vurguluyor; **daemon**'ın içinde artık konteynır yaratacak tek bir satır kod bile yoktur.
* **daemon** ile **containerd** arasındaki iletişim, **gRPC** üzerinden çalışan **CRUD** tarzı bir API ile yapılır.

### 3. containerd ve runc İş Birliği

* İsmine rağmen **containerd** de aslında konteynır yaratamaz.
* **containerd**, Docker imajını bir **OCI bundle** (OCI paketi) haline getirir ve **runc**'a bu paketi kullanarak konteynırı yaratmasını söyler.

### 4. Final: Kernel Seviyesinde Yaratım

* **runc**, işletim sistemi çekirdeğiyle (**kernel**) doğrudan iletişime geçer.
* Konteynırı oluşturmak için gerekli olan **namespaces** ve **cgroups** gibi yapıları bir araya getirir.
* Konteynır, **runc**'ın bir **child process**'i (alt süreç) olarak başlar.
* Konteynır tam olarak başladığı anda, **runc** işini bitirir ve sistemden çıkar (**exits**).

---

**Özetle:** Bir konteynırın doğuşu; bir emrin (**CLI**), bir yöneticiye (**daemon**), oradan bir hazırlıkçıya (**containerd**) ve en son işi yapan bir ustaya (**runc**) iletilmesiyle gerçekleşir.

Bu bölüm, Docker mimarisinin en zekice parçalarından biri olan ve "ara parça" görevi gören **shim** bileşenini anlatıyor. Yazar, bu küçük ama güçlü yapının neden var olduğunu ve sağladığı üç ana faydayı açıklıyor:

---

## Shim Nedir ve Ne İşe Yarar?

Shim aslında çok kritik görevler üstlenir. Bir konteynır başlatıldığında, `containerd` her konteynır için yeni bir **shim** ve bir `runc` süreci (process) başlatır.

### 1. Daemonless Containers (Daemon'dan Bağımsız Konteynırlar)

En büyük avantajlardan biri budur. Eskiden Docker daemon çökerse veya güncellenirse tüm konteynırlar dururdu.

* **Shim sayesinde:** Docker daemon'ı durdurabilir, yeniden başlatabilir ve hatta güncelleyebilirsiniz; bu sırada çalışan konteynırlar bundan hiç etkilenmez.

### 2. Gelişmiş Verimlilik (Improved Efficiency)

Konteynır ayağa kalktığında işleyiş şöyledir:

* `runc` süreci, konteynır çalışmaya başladığı anda görevini bitirir ve sistemden çıkar (exit).
* Geriye sadece çok hafif (lightweight) olan **shim** süreci kalır ve konteynırın yeni "ebeveyn süreci" (parent process) olur.
* **Shim'in görevleri:** Konteynırın durumunu raporlamak ve **STDIN** / **STDOUT** (giriş-çıkış) akışlarını açık tutmak gibi düşük seviyeli işleri yönetmektir.

### 3. Değiştirilebilir (Pluggable) OCI Katmanı

Shim yapısı, Docker'ı esnek hale getirir.

* Varsayılan olarak kullanılan `runc` yerine, başka bir **low-level runtime** (alt seviye çalışma zamanı) kullanmak isterseniz, shim katmanı sayesinde bunu kolayca sisteme entegre edebilirsiniz.

---

**Özetle:** Shim, konteynırın "bakıcısıdır". `runc` konteynırı doğurur ve gider, `containerd` ise yukarıdan her şeyi izler; ancak o konteynırın elinden tutan ve onun daemon'dan bağımsız yaşamasını sağlayan asıl parça **shim**'dir.


