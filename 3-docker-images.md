# Docker Image'ları

## Docker İmajları: Kısa Özet (TLDR)

Yazar, konuya girmeden önce önemli bir not düşüyor: **Image**, **Docker image**, **container image** ve **OCI image** terimlerinin hepsi aynı şeyi ifade eder ve kitap boyunca birbirinin yerine kullanılacaktır.

### 1. İmaj Nedir? (En Basit Tanımıyla)

İmaj, bir uygulamayı çalıştırmak için ihtiyacınız olan her şeyi içeren **salt okunur (read-only)** bir pakettir.

* **İçinde ne var?** Uygulama kodu, bağımlılıklar (dependencies), temel işletim sistemi yapıları ve metadata.
* **Çoklu kullanım:** Tek bir imajdan dilediğiniz kadar (yüzlerce, binlerce) konteynır başlatabilirsiniz.

### 2. Benzetmelerle İmaj Kavramı

Yazar, imajı farklı disiplinlerden gelenler için iki şekilde somutlaştırıyor:

* **Sistem Yöneticileri İçin (VMware):** İmajlar, **VM template**'lerine (şablonlarına) benzer. Şablon durdurulmuş bir sanal makine gibidir; imaj da durdurulmuş bir konteynır gibidir.

### 3. İmajları Elde Etmek ve Depolamak

* **Pull Etmek:** Bir imajı kullanmanın en kolay yolu, onu bir **registry**'den (kayıt dizini) kendi makinenize "çekmektir" (pull).
* **Docker Hub:** En popüler registry'dir. Ancak Docker, diğer tüm registry'lerle de sorunsuz çalışır.

### 4. Katmanlı Yapı (Layers) – Kritik Konsept

Docker imajları, bağımsız katmanların üst üste binmesiyle oluşur:

* **Örnek:** En altta işletim sistemi katmanı, onun üstünde uygulama bağımlılıkları, en üstte ise uygulamanın kendisi bulunur.
* **Birleşik Görünüm:** Docker bu katmanları üst üste dizer (layers) ve size sanki tek bir bütünmüş (unified object) gibi sunar.

### 5. Boyutlar

İmajlar genellikle çok küçük olacak şekilde optimize edilir:

* **NGINX:** Yaklaşık 80MB.
* **Redis:** Yaklaşık 40MB.
* **Not:** Windows imajları istisnadır; onlar çok büyük olabilir.

---

**Özetle:** İmaj, uygulamanın çalışması için gereken her şeyin katmanlar halinde paketlendiği, dondurulmuş bir kopyasıdır.

---

## İmajlara Giriş 2: Yapı Taşları ve Çalışma Mantığı

* **İmajlar:** Build-time (İnşa zamanı) yapılarıdır. Uygulamanın dondurulmuş halidir.
* **Konteynırlar:** Run-time (Çalışma zamanı) yapılarıdır. İmajın hayata geçmiş halidir.

### 1. İmaj ve Konteynır Arasındaki Bağ

Bir imajdan konteynır başlattığınızda (genellikle `docker run` ile), bu ikisi birbirine "bağlanır".

* **Silme Kuralı:** Çalışan bir konteynırın kullandığı imajı silemezsiniz.
* **Zincirleme Etki:** Eğer bir imajdan 10 tane konteynır oluşturduysanız, o imajı silebilmek için önce bu 10 konteynırın tamamını silmeniz gerekir.

### 2. "Sadece Gerektiği Kadar" İşletim Sistemi

Konteynırlar tek bir uygulama veya mikroservis çalıştırmak için tasarlanmıştır. Bu yüzden imajlar sadece **mutlak gerekli** olanları içermelidir:

* **Gereksizleri Ayıklamak:** Derleme araçları (build tools) veya hata ayıklama araçları imajda yer almamalıdır.
* **Slim (İnce) İmajlar:** Örneğin **Alpine Linux** imajı sadece **3MB** civarındadır. İçinde 10 yılda bir lazım olacak araçlar, paket yöneticileri veya birden fazla kabuk (shell) bulunmaz. Eğer uygulama ihtiyaç duymuyorsa, imajda bir kabuk bile olmayabilir.

### 3. Neden Bu Kadar Küçükler? (Kernel Farkı)

İmajların küçük olmasının en büyük sırrı içlerinde **işletim sistemi çekirdeği (kernel)** bulunmamasıdır.

* Konteynırlar, üzerinde çalıştıkları **host makinenin kernel'ını paylaşırlar**.
* İmajın içindeki işletim sistemi bileşenleri sadece dosya sistemi nesnelerinden ibarettir. Buna "just enough OS" (tam yettiği kadar işletim sistemi) denir.

### 4. Windows İstisnası

Linux dünyasında megabaytlarla konuşurken, Windows tarafında durum farklıdır. Windows tabanlı imajlar gigabaytlarca büyüklükte olabilir, bu da onların internet üzerinden çekilmesini (pull) ve itilmesini (push) oldukça yavaşlatır.

---

**Özetle:** İmajlar sadece uygulamanın çalışması için gereken minimum seti içeren, host kernel'ını kullanan ve konteynır silinmeden silinemeyen paketlerdir.

---

## İmajları Çekmek (Pulling Images)

Yeni kurulmuş bir Docker sisteminde yerel depo (**local repository**) boştur. Yerel depo, Docker'ın imajları daha hızlı erişim için sakladığı bir alandır (buna bazen **image cache** de denir).

* **Linux'ta Konum:** Genellikle `/var/lib/docker/<storage-driver>` altındadır.
* **Docker Desktop'ta:** Docker VM'in içindedir.

### 1. Varsayılan Kabuller (Docker'ın "Önyargıları")

Bir imajı sadece ismiyle çektiğinizde (örneğin: `docker pull redis`), Docker senin yerine iki varsayımda bulunur:

1. **Etiket (Tag):** Senin `latest` (en güncel) sürümü istediğini varsayar.
2. **Kaynak (Registry):** İmajı **Docker Hub** üzerinden çekmek istediğini varsayar (`docker.io`).
*Bu varsayılanları komut satırında imaj adını tam yazarak değiştirebilirsin.*

### 2. Katmanlı İndirme ve Verimlilik

* Sistemde yüklü olan başka bir imaj, Redis ile **aynı ortak katmanı** kullanıyor ise o katman pull redis dendiğinde yeniden indirilmez. Bir katman dosya sisteminde sadece bir defa indirilir ve bu katmanı birden fazla image kullanabilir.

### 3. Neden Bu Çok Önemli?

Docker'ın bu zekası sayesinde:

* **Bant Genişliği Tasarrufu:** Aynı katmanı tekrar tekrar internetten indirmezsin.
* **Disk Tasarrufu:** Aynı katman diskte sadece **bir kez** saklanır ancak yüzlerce farklı imaj tarafından ortaklaşa kullanılır.
* **Hız:** Yeni bir imaj çekerken, eğer temel katmanlar (örneğin Ubuntu veya Alpine katmanı) zaten makinenizdiyse, sadece uygulamanıza özel olan üst katmanlar indirilir ve işlem saniyeler içinde biter.

---

**Özetle:** İmaj çekme işlemi, bir imajın tüm katmanlarını değil, sadece elinde olmayan katmanlarını indirme sürecidir. Yerel depoda (`docker images`) gördüğün imajlar aslında birbirine "geçmeli" bir yapboz gibi ortak parçaları kullanır.

---

## İmaj Kayıt Dizinleri (Image Registries)

İmajlar, farklı ortamlardan (geliştirme, test, üretim) kolayca erişilebilmesi için **Registry** adı verilen merkezî yerlerde depolanır.

### 1. Standartlar ve Uyumluluk

Modern registry'lerin çoğu iki temel standardı destekler:

* **OCI distribution-spec:** İmajların nasıl dağıtılacağını belirleyen evrensel standart.
* **Docker Registry v2 API:** Docker CLI gibi araçların registry ile standart bir şekilde konuşmasını sağlayan arayüz.
* **Gelişmiş Özellikler:** Bazı registry'ler sadece depolama yapmakla kalmaz; imaj tarama (güvenlik açığı kontrolü) ve CI/CD süreçleriyle entegrasyon gibi ek özellikler sunar.

### 2. Registry Çeşitleri

* **Docker Hub:** En yaygın kullanılan genel (public) registry. Docker, aksini belirtmediğiniz sürece burayı varsayılan kabul eder.
* **Üçüncü Taraf ve Yerel Çözümler:** Kurumların kendi iç ağlarında barındırdığı (on-premises) veya bulut sağlayıcıların (AWS ECR, Google GCR, Azure ACR) sunduğu özel registry'ler de mevcuttur.

### 3. Hiyerarşik Yapı (Registry > Repository > Image)

Yazar, registry yapısını bir kütüphaneye benzetebileceğimiz şu hiyerarşiyle açıklıyor:

1. **Registry (Kütüphane):** En üst katman. (Örn: Docker Hub)
2. **Repository (Kitaplık):** Belirli bir uygulamanın tüm sürümlerini içeren bölüm. (Örn: `redis` veya `nginx` klasörü)
3. **Image (Kitap):** Belirli bir etikete (tag) veya kimliğe (digest) sahip olan fiziksel dosya. (Örn: `redis:latest`, `redis:7.0`)

---

**Özetle:** Registry bir depo sitesidir; Repository o site içindeki bir projenin klasörüdür; Image ise o klasörün içindeki her bir farklı sürümdür.

---

## Resmî Depolar (Official Repositories)

Docker Hub, Docker ekibi ve uygulama geliştiricileri tarafından bizzat doğrulanmış ve bakımı yapılan imajlar için özel bir alan sunar.

### 1. Neden Resmî Depoları Tercih Etmelisiniz?

Resmî depolar şu avantajları sağlar:

* **Güncellik:** En son yamaları ve özellikleri içerirler.
* **Güvenlik:** Docker tarafından taranmış ve onaylanmış kodlardır.
* **Dokümantasyon:** Nasıl kullanılacaklarına dair temiz ve detaylı rehberler sunarlar.
* **En İyi Uygulamalar (Best Practices):** İmajın boyutu ve performansı optimize edilmiştir.

### 2. Nasıl Tanınırlar?

Resmî depoları diğerlerinden ayıran iki belirgin özellik vardır:

* **Yeşil Rozet:** Üzerinde "Docker Official Image" yazan yeşil bir rozet taşırlar.
* **Kısa URL/Namespace:** Docker Hub hiyerarşisinin en üstünde yer alırlar. URL yapılarında bir kullanıcı adı bulunmaz (örneğin `user/nginx` yerine sadece `_` işareti ile `/_/nginx/` olarak görünürler).

**Örnekler:**

* `nginx`
* `redis`
* `mongo`
* `alpine`

### 3. Çoklu Mimari Desteği (Multi-Arch)

- Bu imajların (örneğin Alpine veya NGINX) genellikle sadece standart PC'ler (x86) için değil, çok geniş bir CPU yelpazesi (ARM, s390x vb.) için hazırlandığını vurguluyor. Yani aynı imaj adını kullanarak hem bir MacBook M3'te hem de bir Intel sunucuda sorunsuzca çalışabilirsiniz.

---

**Özetle:** Resmî depolar, Docker ekosisteminin "altın standartlarıdır". Bir imaj ararken ilk bakmanız gereken yer burasıdır.

---

## İmaj İsimlendirme ve Etiketleme (Naming & Tagging)

Docker'da bir imajın ismine bakarak onun nereden geldiğini, kime ait olduğunu ve hangi sürüm olduğunu anlayabilirsiniz. Bu **"Tam Nitelikli İmaj Adı" (Fully Qualified Image Name)** olarak tanımlanır.

### 1. İmaj Adının Anatomisi

Tam bir imaj adı şu dört parçadan oluşur:
**`[Registry Adı] / [Kullanıcı/Organizasyon Adı] : [Repository Adı] : [Etiket (Tag)]`**

Eğer siz bazı kısımları yazmazsanız, Docker bunları otomatik doldurur:

* **Registry:** Belirtilmezse `docker.io` (Docker Hub) kabul edilir.
* **Etiket:** Belirtilmezse `latest` kabul edilir.

### 2. Resmî Depolardan İmaj Çekmek

Resmî imajlarda kullanıcı adı kısmı boştur (veya `library` olarak geçer). Yazımı çok basittir:
`docker pull <repository>:<etiket>`

**Örnekler:**

* `docker pull redis:8.0-M02` (Spesifik bir sürüm)
* `docker pull busybox:glibc` (Farklı bir kütüphane varyasyonu)
* `docker pull alpine` (Etiket yazılmadığı için otomatik `latest` olur)

### 3. "latest" Etiketi Hakkında Kritik Uyarılar

* **Garanti Değildir:** `latest` etiketli bir imaj, her zaman o depodaki en güncel imaj olmayabilir. Sadece "latest" ismi verilmiş bir etikettir.
* **Hata Riski:** Eğer bir depoda `latest` isimli bir etiket oluşturulmamışsa, etiketsiz çekme komutu hata verir.

### 4. Kullanıcı Depolarından ve Farklı Registry'lerden Çekmek

* **Kullanıcı Deposu:** Başına kullanıcı adını eklersiniz.
* `docker pull nigelpoulton/tu-demo:v2`

* **Farklı Registry:** En başa registry'nin DNS adını eklersiniz (Örn: GitHub Container Registry).
* `docker pull ghcr.io/regclient/regsync:latest`

### 5. Standartların Gücü

Farklı bir registry'den (GHCR gibi) imaj çekerken bile sürecin tamamen aynı görünmesinin nedeni, bu registry'lerin de **OCI registry-spec** ve **Docker Registry v2 API** standartlarını uygulamasıdır.

---

**Özetle:** Bir imajın adı, onun internetteki tam adresidir. Adresi ne kadar detaylı yazarsanız (etiket ve registry dahil), o kadar tutarlı sonuçlar alırsınız.

---

## Çoklu Etiketli İmajlar (Images with Multiple Tags)

Docker'da bir imajın (fiziksel dosya bütünlüğü) birden fazla ismi olabilir. Bu, bir dosyaya birden fazla "kısayol" oluşturmaya benzer.

### 1. Aynı İmaj ID, Farklı İsimler

Yazarın verdiği örneği incelediğimizde:

* `nigelpoulton/tu-demo:latest` -> Image ID: `b4210d0aa52f`
* `nigelpoulton/tu-demo:v1`     -> Image ID: `b4210d0aa52f`
* `nigelpoulton/tu-demo:v2`     -> Image ID: `6ba12825d092`

Burada aslında **3 imaj değil, 2 imaj** var. `latest` ve `v1` etiketleri tamamen aynı imajı (`b4210d0aa52f`) işaret ediyor.

### 2. "latest" Etiketinin Kandırmacası

Bu örnek, daha önce yapılan uyarıyı kanıtlıyor: **`latest` etiketi her zaman en yeni imaj demek değildir.**

* Örnekte `v2` etiketi sadece 12 dakika önce oluşturulmuş (en yeni olan o).
* Ancak `latest` etiketi, 2 gün önce oluşturulmuş olan `v1` imajına takılı kalmış.

**Sonuç:** Eğer bir geliştirici imajı güncelledikten sonra `latest` etiketini manuel olarak yeni imaj ID'sine kaydırmazsa (re-tag), `latest` etiketi eski bir sürümü işaret etmeye devam eder.

---

**Özetle:** Bir imajın kaç tane ismi (etiketi) olduğu önemli değildir; asıl kimliği **Image ID**'sidir. `latest` ismine asla körü körüne güvenmemeli, her zaman Image ID veya spesifik versiyon numaralarını kontrol etmelisiniz.

Yazarın bu bölümde paylaştığı tüm teknik detayları, komut çıktılarını ve kritik uyarıları kapsayan **eksiksiz** özeti aşağıda bulabilirsin:

---

## İmajlar ve Katmanlar (Images and Layers) - Eksiksiz Teknik İnceleme

Docker imajları, birbirine gevşek şekilde bağlı, **salt-okunur (read-only)** katmanlar koleksiyonudur. Her bir katman, bir veya daha fazla dosyadan oluşur. Docker bu katmanları üst üste istifler (stacking) ve kullanıcıya tek bir birleşik dosya sistemi (unified image) olarak sunar.

Docker bize bir imajın katman bilgilerini incelemek için **üç temel yöntem** sunmaktadır:

### 1. Pull İşlemi (Canlı Takip)

Bir imajı çekerken Docker'ın katmanları nasıl tek tek ele aldığını görebilirsiniz.

* **Komut:** `docker pull node:latest`
* **Çıktı Analizi:** Her bir `Pull complete` ile biten satır, Docker tarafından çekilen bağımsız bir katmanı temsil eder.
* **Örnek Çıktı Satırları:**
* `952132ac251a: Pull complete`
* `82659f8f1b76: Pull complete`
* ... (toplam 5 katman)


* **Gözlem:** Bu çıktıdaki kısa ID'ler, katmanların o anki transfer kimlikleridir.

### 2. `docker inspect` (Image'ın Derinlemesine Yapısal Analizi)

İmajın tüm metadata ve katman bilgilerini JSON formatında görmek için kullanılır.

* **Komut:** `docker inspect node:latest`
* **Önemli Bölüm (RootFS):** Çıktının sonlarına doğru yer alan `RootFS` kısmı, imajın fiziksel katmanlarını listeler.
* **JSON Yapısı:**
```json
"RootFS": {
    "Type": "layers",
    "Layers": [
        "sha256:c8a75145fc...",
        "sha256:c6f2b330b6...",
        ...
    ]
}

```


* **Kritik Not:** Buradaki SHA256 hash'leri, `docker pull` çıktısındaki kısa ID'lerden farklıdır. Çünkü `inspect`, katmanın içeriğine dayalı değişmez karma kodlarını (content hashes) gösterir.

### 3. `docker history` (İnşa Geçmişi ve Katman Evrimi)

İmajın nasıl oluşturulduğunu, hangi Dockerfile talimatlarının hangi katmanlara yol açtığını gösterir.

* **Komut:** `docker history node:latest`
* **Analiz:** Bu komut imajın "soy ağacını" çıkarır. Ancak burada önemli bir ayrım var: **"Her satır fiziksel bir katman değildir."**
* **Metadata vs. Layer:** - `ENV`, `EXPOSE`, `CMD`, `ENTRYPOINT` gibi talimatlar imaja sadece bilgi (metadata) ekler.
* Bu talimatlar `history` çıktısında görünse de, karşılarında **0B** (sıfır bayt) yazar, yani dosya sisteminde yeni bir fiziksel katman oluşturmazlar.
* Sadece dosya sisteminde değişiklik yapan (dosya ekleyen, silen, güncelleyen) talimatlar gerçek katmanlar oluşturur.

---

**Bölümün Teknik Özeti:**
İmajlar statiktir (değişmez). Bir imajı incelediğinizde gördüğünüz katmanlar, o imajın "genetik kodudur". `pull` ile indirmeyi, `inspect` ile içeriği, `history` ile de geçmişi görürsünüz.

---

## Temel Katmanlar ve Katman İstifleme (Base Layers & Stacking)

Docker imajları boşlukta oluşmaz; her zaman bir temel katmanla başlar ve üzerine eklenen her içerik yeni bir katman oluşturur.

### 1. İmaj İnşa Süreci (Örnek Senaryo)

1. **Katman 1 (Base Layer):** Kurumsal politika gereği seçilen resmî **Ubuntu 24.04** imajı.
2. **Katman 2:** Bu işletim sistemi üzerine kurulan onaylı **Python** sürümü.
3. **Katman 3:** Uygulamanın kendi **kaynak kodu**.
*Sonuç:* Üç katmanlı, bağımsız parçalardan oluşan ancak tek bir paket gibi davranan bir imaj.

### 2. İmajın Kimliği: "Bağımsız Nesneler + Metadata"

Çok kritik bir teknik detay: **İmaj aslında fiziksel bir "dosya" değildir.**

* Katmanlar, disk üzerinde birbirlerinden tamamen **bağımsız nesneler** olarak saklanır.
* İmaj ise sadece bu katmanların hangileri olduğunu ve hangi sırayla üst üste dizilmesi gerektiğini söyleyen bir **metadata (bilgi)** dosyasıdır.

### 3. Dosyaların Üst Üste Binmesi (Obscuring/Shadowing)

Bir üst katmandaki dosya, alt katmandaki aynı isimli dosyayı nasıl etkiler?

* **Senaryo:** Alt katmanda `Dosya 5` var. Üst katmana bu dosyanın güncellenmiş hali olan `Dosya 7` (aynı isim ve konumda) eklendi.
* **Görünüm:** Üstteki katman, alttaki dosyayı **maskeler (obscure)**. Kullanıcı konteynır içinden baktığında sadece en üstteki (güncel) dosyayı görür.
* **Güncelleme Mantığı:** Docker'da bir dosyayı güncellemek, eski katmandaki dosyayı değiştirmek değil; **değişikliği içeren yeni bir katmanı en üste eklemektir.**

### 4. Birleşik Görünüm ve Storage Driver (Depolama Sürücüleri)

Bu katmanlı ve parçalı yapıyı bizim için "tek bir disk" gibi birleştiren teknolojiye **Storage Driver** denir.

* **Standart:** Modern sistemlerin neredeyse tamamı **`overlay2`** sürücüsünü kullanır.
* **Alternatifler:** `zfs`, `btrfs` ve `vfs` gibi seçenekler de mevcuttur.
* **Sonuç:** Hangi sürücü kullanılırsa kullanılsın, kullanıcı deneyimi değişmez; tüm katmanlar birleşir ve ortaya **"Unified View" (Birleşik Görünüm)** çıkar. Konteynır içindeki uygulama, bu katmanların ayrı olduğunu fark etmez bile.

---

**Özetle:** İmajlar, üst üste binmiş şeffaf asetat kağıtları gibidir. Her kağıt (katman) bağımsızdır ancak üst üste konulup ışığa tutulduğunda (Storage Driver ile mount edildiğinde) karşınıza tek bir resim (dosya sistemi) çıkar.


---

## Digest (Özet) ile İmaj Çekmek

Şimdiye kadar imajları isimleri (etiketleri) üzerinden çektik. Bu yöntemde ciddi bir problem vardır: **Etiketler mutabıldır (değişebilir).**

### 1. Değişebilir Etiketlerin (Mutable Tags) Riski

Etiketler keyfidir ve her an başka bir imajı işaret edecek şekilde değiştirilebilir.

* **Problem:** Bugün çektiğiniz `alpine:latest` imajı ile bir yıl önce çektiğiniz aynı etiketli imaj muhtemelen aynı değildir.
* **Kritik Hata Senaryosu:** `golftrack:1.5` isimli bir imajınız olduğunu ve içinde güvenlik açığı bulunduğunu düşünün. Hatayı düzeltip yeni imajı **aynı etiketle** (`1.5`) sisteme yüklerseniz (overwrite), elinizde hangi konteynırın eski (güvensiz) hangisinin yeni (güvenli) olduğunu anlamanın bir yolu kalmaz. Her ikisinin de ismi aynıdır!

### 2. Çözüm: Digest (Content Addressable Storage)

Docker, **İçerik Adreslemeli Depolama** modelini kullanır. Bu modelde her imajın, içeriğinden üretilen kriptografik bir hash değeri (SHA256) vardır. Buna **Digest** denir.

* **Değişmezlik:** İmajın içinde tek bir bit bile değişse, digest tamamen değişir.
* **Çakışmazlık:** İki farklı imajın aynı digest'e sahip olması imkansızdır.
* **Güvenlik:** Digest kullandığınızda, tam olarak istediğiniz içeriği indirdiğinizden %100 emin olursunuz.

### 3. Digest Değerini Nasıl Buluruz?

Digest değerini öğrenmek için üç farklı yöntem sunuluyor:

* **Yerel İmajlar İçin:**
`$ docker images --digests alpine`
(Bu komut, yerel depodaki imajın `sha256:...` ile başlayan uzun özetini gösterir.)
* **İndirmeden Önce (Docker CLI ile):**
`$ docker buildx imagetools inspect nigelpoulton/k8sbook:latest`
(Registry'deki imajın digest değerini çekip getirir.)
* **API Üzerinden (curl ile):**
`$ curl "https://hub.docker.com/v2/repositories/nigelpoulton/k8sbook/tags/?name=latest" | jq '.results[].digest'`

### 4. Digest ile İmaj Çekme Komutu

Bir imajı digest ile çekmek için iki nokta (`:`) yerine ampersand (`@`) sembolü kullanılır:

`$ docker pull nigelpoulton/k8sbook@sha256:13dd59a0...bce2e14b`

---

**Özetle:** Üretim (production) ortamlarında, imajın içeriğinin değişmediğinden ve her zaman aynı kodun çalıştığından emin olmak için etiketler yerine **Digest** kullanmak en iyi uygulama (best practice) olarak kabul edilir.

Tabii 👍 işte **kısa, net ve akılda kalıcı bir özet**:

---

## Docker Digest Hash Mekanizması 

* Docker imajı tek parça değildir; **manifest + bağımsız katmanlardan** oluşur.

* Bu yüzden **iki ana hash türü** vardır:

  * **Image Digest:** İmajın manifest’inin hash’i (imajın kimliği)
  * **Layer Digest:** Katmanın içeriğinin hash’i

* Katmanlar registry’ye gönderilirken **sıkıştırılır**.

* Sıkıştırma dosyanın bit yapısını değiştirdiği için **tek hash yeterli değildir**.

* Bu nedenle her katman için **iki ayrı hash** tutulur:

  * **Content Hash:** Katmanın sıkıştırılmamış, gerçek içeriği (diskteki doğruluk)
  * **Distribution Hash:** Katmanın sıkıştırılmış, ağdan geçen hali (transfer güvenliği)

* Farklı komutlarda farklı hash görmen:

  * **Hata değil**
  * Docker’ın bilinçli tasarımıdır

### Sonuç

Docker bu çift hash sistemi sayesinde:

* Verinin **değişmediğini**
* **Bozulmadığını**
* **Transfer sırasında kurcalanmadığını**

%100 güvenilir şekilde garanti eder.

---


