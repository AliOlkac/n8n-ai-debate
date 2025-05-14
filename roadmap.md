# n8n ile LLM Münazara Sistemi Proje Yol Haritası

Bu yol haritası, n8n üzerinde iki LLM'in münazara yapacağı bir sistem oluşturma sürecinde size rehberlik edecektir.

## Adım 1: n8n Kurulumu ve Arayüze Giriş

### 1.1. n8n Kurulum Seçenekleri
    - **n8n Cloud:** En hızlı başlangıç için n8n'in bulut versiyonunu kullanabilirsiniz. Kayıt olup hemen başlayabilirsiniz.
    - **Local Kurulum (Docker):** Kendi bilgisayarınızda veya sunucunuzda n8n'i Docker ile kurabilirsiniz. Bu size daha fazla kontrol sağlar.
        - Docker Desktop'ın bilgisayarınızda kurulu olduğundan emin olun.
        - Terminal veya PowerShell'i açın.
        - `docker run -it --rm --name n8n -p 5678:5678 n8nio/n8n` komutunu çalıştırın.
        - Tarayıcınızda `http://localhost:5678` adresine gidin.
    - **npm ile Kurulum:** Node.js ve npm kurulu ise `npm install n8n -g` komutu ile global olarak kurup `n8n` komutu ile başlatabilirsiniz.

### 1.2. n8n Arayüzüne İlk Bakış
    - **Workflow Canvas (İş Akışı Tuvali):** Ortadaki ana alan, nodları (düğüm) sürükleyip bırakarak iş akışınızı oluşturacağınız yerdir.
    - **Nodes Panel (Düğüm Paneli):** Sol tarafta, kullanabileceğiniz tüm entegrasyonları (API'ler, servisler) ve yardımcı nodları bulabilirsiniz. Arama çubuğunu kullanarak istediğiniz nodu hızlıca bulabilirsiniz.
    - **Parameters Panel (Parametre Paneli):** Bir nod seçildiğinde sağ tarafta açılır. Bu panelde seçili nodun ayarlarını, kimlik bilgilerini ve giriş/çıkış verilerini yapılandırabilirsiniz.
    - **Execute Workflow (İş Akışını Çalıştır):** Sağ alttaki "Execute Workflow" butonu ile oluşturduğunuz akışı test edebilirsiniz.

### 1.3. İlk İş Akışını Oluşturma (Workflow)
    - "Add workflow" butonu ile yeni bir iş akışı oluşturun.
    - İlk olarak bir "Start" nodu otomatik olarak eklenir. Bu, akışın başlangıç noktasıdır.

## Adım 2: Münazara Konusunu Kullanıcıdan Alma

### 2.1. Manuel Tetikleme ve Form Oluşturma
    - **Manual Trigger (Manuel Tetikleyici) / Start Node:** Akışın manuel olarak başlatılması için kullanılır.
    - **User Input (Kullanıcı Girişi) Alternatifi:** Eğer akışı başlattığınızda kullanıcıdan bir konu girmesini istiyorsanız, n8n'de doğrudan bir "form" nodu başlangıçta bulunmaz. Bunun yerine, akışı tetikleyecek bir Webhook nodu kullanabilir ve bu webhook'a bir HTML formu ile veri gönderebilirsiniz. Ya da daha basit bir başlangıç için, konuyu bir "Set" nodu ile manuel olarak tanımlayabilirsiniz.
    - **Örnek (Set Nodu ile Konu Belirleme):**
        - Nodes panelinden "Set" nodunu aratın ve tuvale sürükleyin.
        - Start nodunun çıkışını Set nodunun girişine bağlayın.
        - Set noduna tıklayın. Parameters panelinde "Keep Only Set" seçeneğini işaretleyin.
        - "Add Value" butonuna tıklayın.
        - "Name" alanına `konu` yazın.
        - "Value" alanına örnek bir münazara konusu yazın (örn: "Yapay zekanın gelecekteki iş gücüne etkisi").
        - Akışı çalıştırarak Set nodunun çıktısını kontrol edin.

## Adım 3: LLM (Gemma 2.0 Flash) API Entegrasyonu

### 3.1. LLM API Key Edinme
    - Kullanacağınız LLM servisinin (örneğin Google AI Studio for Gemma, veya başka bir LLM sağlayıcısı) web sitesine gidin.
    - Bir hesap oluşturun ve API anahtarınızı (API Key) edinin. Bu anahtar, n8n'in LLM ile iletişim kurması için gereklidir.

### 3.2. n8n'de HTTP Request Nodu ile LLM'e Bağlanma
    - Nodes panelinden "HTTP Request" nodunu aratın ve tuvale sürükleyin.
    - Set nodunun (veya konu girişini yaptığınız nodun) çıkışını HTTP Request nodunun girişine bağlayın.
    - **HTTP Request Nodu Ayarları (LLM 1 için):**
        - **Request Method:** `POST` (genellikle LLM API'leri için POST kullanılır).
        - **URL:** LLM API'nizin "endpoint" URL'sini girin. (Örn: Gemma için `https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent?key=YOUR_API_KEY` - `YOUR_API_KEY` kısmını kendi anahtarınızla değiştirin).
        - **Authentication:** Gerekirse (API dokümantasyonuna bakın), "Header Auth" veya "Generic Credential" seçeneğini kullanın. API anahtarınızı genellikle "Authorization" header'ında `Bearer YOUR_API_KEY` formatında veya API'ye özel bir header ile gönderirsiniz. Google Gemma için API anahtarı URL'ye eklenir.
        - **Body Content Type:** `JSON`.
        - **Specify Body:** `Using JSON`
        - **JSON (Body):** LLM API'nizin beklediği formatta JSON verisini girin. Bu genellikle prompt'u, model ayarlarını (temperature, max tokens vb.) içerir.
            ```json
            // Örnek Gemma API isteği (model ve prompt'u kendinize göre uyarlayın)
            {
              "contents": [{
                "parts":[{
                  "text": "Sen A kişisisin. Şu konuda bir argüman sun: {{ $json.konu }}. Argümanın kısa ve net olsun."
                }]
              }]
            }
            ```
            - `{{ $json.konu }}` ifadesi, bir önceki Set nodundan gelen "konu" verisini buraya dinamik olarak ekler. Expression Editor'ü kullanarak mevcut verilere göz atabilirsiniz.
        - **Options > Response Format:** `JSON`.
        - **Options > Split Into Items:** API yanıtı bir liste içeriyorsa ve her bir öğeyi ayrı bir n8n öğesi olarak işlemek istiyorsanız bunu aktif edin. Genellikle LLM yanıtları için kapalı tutulur.
    - Bu nodu kopyalayıp yapıştırarak ikinci LLM için de benzer bir HTTP Request nodu oluşturun. Sadece prompt'u ve belki kişilik tanımını değiştirin.

## Adım 4: Promptların ve Kişiliklerin Tasarlanması

### 4.1. Kişilik Tanımları
    - Her LLM için ayrı bir kişilik (persona) tanımlayın. Örneğin:
        - **LLM 1 (Tartışmacı A):** Konuyu [X] perspektifinden savunan, iddialı ve kanıt odaklı.
        - **LLM 2 (Tartışmacı B):** Konuyu [Y] perspektifinden ele alan, sorgulayıcı ve uzlaşmacı.
### 4.2. Prompt Mühendisliği
    - **İlk Argüman Promptları:**
        - LLM 1 için: "Sen [Kişilik A tanımı]. Münazara konumuz: '{{ $json.konu }}'. Bu konuda ilk argümanını sun."
        - LLM 2 için: "Sen [Kişilik B tanımı]. Münazara konumuz: '{{ $json.konu }}'. [Tartışmacı A'nın ilk argümanı buraya eklenecek]. Bu argümana karşı cevabını ve kendi ilk argümanını sun."
    - **Cevap Promptları:**
        - Bir LLM'in cevabını diğer LLM'e gönderirken, prompt'a önceki konuşmayı dahil edin.
        - Örnek: "Sen [Kişilik A tanımı]. Karşı tarafın son argümanı: '{{ $json.llm2_cevabi }}'. Bu argümana cevabını ve yeni argümanını sun."

## Adım 5: Sıralı Münazara Akışının n8n'de Kurulması

### 5.1. Döngü Yapısı (Looping)
    - n8n'de doğrudan "for" veya "while" döngüsü nodu yoktur. Döngüsel davranışlar genellikle "SplitInBatches" nodu, "Merge" nodu ve bir IF nodu ile bir koşul sağlanana kadar akışın bir kısmını tekrar çalıştırarak oluşturulur.
    - **Basit Sıralı Akış (Döngüsüz Başlangıç - Belirli Sayıda Tur):**
        1. **Start Node**
        2. **Set Node (Konu Belirleme)**
        3. **HTTP Request (LLM 1 - İlk Argüman)**
            - Bu nodun çıktısındaki metni bir sonraki adıma aktarmak için "Set" nodu kullanabilirsiniz. Örneğin, Set nodunda `llm1_arguman1` adında bir değer oluşturup HTTP Request nodunun yanıtındaki metni buna atayın.
        4. **HTTP Request (LLM 2 - Cevap ve Argüman)**
            - Prompt'una `{{ $json.llm1_arguman1 }}` ekleyin.
            - Çıktısını `llm2_cevap1` olarak Set nodu ile kaydedin.
        5. **HTTP Request (LLM 1 - Cevap ve Argüman)**
            - Prompt'una `{{ $json.llm2_cevap1 }}` ekleyin.
            - Çıktısını `llm1_arguman2` olarak Set nodu ile kaydedin.
        6. Bu şekilde birkaç tur manuel olarak devam ettirebilirsiniz.

### 5.2. Veri Aktarımı ve Birleştirme
    - Her LLM yanıtından sonra, önemli bilgileri (üretilen metin, konuşmacı adı vb.) bir "Set" nodu ile düzenli bir yapıda saklayın.
    - Eğer tüm konuşmaları birleştirmek isterseniz, her turda üretilen metinleri bir "Function" nodu veya "Edit Fields" nodu ile bir diziye (array) ekleyebilirsiniz.
    - **Function Nodu ile Metinleri Birleştirme Örneği:**
        ```javascript
        // Önceki bir Set nodunda 'tum_konusmalar' adında bir dizi başlatıldığını varsayalım.
        // items[0].json.tum_konusmalar = items[0].json.tum_konusmalar || []; // Eğer dizi yoksa başlat
        // items[0].json.tum_konusmalar.push({
        //   konusmaci: "LLM 1", // veya "LLM 2"
        //   metin: items[0].json.yeni_arguman // LLM'den gelen yeni argüman
        // });
        // return items;

        // Daha basit bir yaklaşım olarak, her LLM yanıtını ayrı bir item olarak tutabilir ve Merge nodu ile en sonda birleştirebilirsiniz.
        // Bu durumda her HTTP Request sonrası bir Set nodu ile konuşmacı bilgisini ve metni ekleyin.
        // Örn: LLM1 sonrası Set: Name: konusmaci, Value: "LLM_A"; Name: metin, Value: {{ $json.yanit_metni_llm1 }}
        // Örn: LLM2 sonrası Set: Name: konusmaci, Value: "LLM_B"; Name: metin, Value: {{ $json.yanit_metni_llm2 }}
        ```

### 5.3. Tur Sayısını Kontrol Etme (İleri Düzey - IF ve Merge ile Döngü)
    - Bu, n8n'de biraz daha karmaşıktır. Genellikle bir "counter" (sayıcı) değişkeni tutulur.
    - **IF Node:** Sayıcı belirli bir değere ulaştı mı diye kontrol eder.
        - True ise döngü biter.
        - False ise akış döngünün başına (veya LLM nodlarına) geri yönlendirilir. Bu yönlendirme, IF nodunun false çıkışını döngünün başındaki bir nodun (genellikle bir Merge nodu) girişine bağlayarak yapılır. Merge nodu, ilk çalıştırmada Start nodundan gelen veriyi, sonraki çalıştırmalarda ise IF nodundan gelen veriyi alır.
    - Şimdilik manuel sıralı akışa odaklanmak daha iyi olabilir. Döngüleri daha sonra ekleyebiliriz.

## Adım 6: ElevenLabs API ile Metinlerin Ses Dosyasına Dönüştürülmesi

### 6.1. ElevenLabs API Key Edinme
    - ElevenLabs web sitesine gidin, hesap oluşturun ve API anahtarınızı alın.
    - Farklı ses seçeneklerini inceleyin.

### 6.2. n8n'de HTTP Request Nodu ile ElevenLabs Entegrasyonu
    - Her LLM yanıtından sonra (veya tüm konuşmalar biriktikten sonra her biri için ayrı ayrı) bir "HTTP Request" nodu ekleyin.
    - **HTTP Request Nodu Ayarları (ElevenLabs için):**
        - **Request Method:** `POST`.
        - **URL:** ElevenLabs API'sinin metinden sese dönüştürme endpoint'i (örn: `https://api.elevenlabs.io/v1/text-to-speech/{voice_id}`). `{voice_id}` kısmını kullanmak istediğiniz sesin ID'si ile değiştirin. Farklı LLM'ler için farklı `voice_id` kullanabilirsiniz.
        - **Authentication:** "Header Auth".
            - "Name": `xi-api-key`
            - "Value": ElevenLabs API anahtarınız.
        - **Body Content Type:** `JSON`.
        - **JSON (Body):**
            ```json
            {
              "text": "{{ $json.llm_yanit_metni }}", // Seslendirilecek metin
              "model_id": "eleven_multilingual_v2", // veya başka bir model
              "voice_settings": {
                "stability": 0.5,
                "similarity_boost": 0.75
              }
            }
            ```
        - **Options > Response Format:** `File`. Bu, API'den dönen ses dosyasını doğrudan n8n içinde binary veri olarak almanızı sağlar.
        - **Options > Response File Name:** Oluşacak ses dosyasının adını belirleyebilirsiniz (örn: `konusma_1.mp3`). Dinamik isimler için expression kullanabilirsiniz: `munazara_tur{{ $json.tur_numarasi }}_llm_A.mp3`.
        - Bu nodun çıktısı, bir sonraki nodda (örneğin "Write Binary File" nodu) kullanılabilecek binary bir ses dosyası olacaktır.

### 6.3. Ses Dosyalarını Kaydetme
    - ElevenLabs'ten dönen binary ses verisini kaydetmek için "Write Binary File" nodunu kullanın.
    - HTTP Request (ElevenLabs) nodunun çıkışını Write Binary File nodunun girişine bağlayın.
    - **Write Binary File Nodu Ayarları:**
        - **File Name:** Ses dosyasının kaydedileceği yolu ve adını belirtin (örn: `/data/munazara/sesler/konusma_1.mp3` - eğer Docker kullanıyorsanız n8n container'ı içindeki bir yoldur bu, dışarıya mount ettiğiniz bir volume'e yazmanız gerekir veya n8n'in kendi kullanıcı verileri klasörüne). Yerel kurulumda direkt bir dosya yolu verebilirsiniz.
        - **Property Name:** Gelen binary verinin hangi özellikte olduğunu belirtir. Genellikle `data` olur. HTTP Request nodunun çıktısını inceleyerek doğru ismi bulun.

## Adım 7: (Opsiyonel) Metinlerin Slayt Şeklinde Video Haline Getirilmesi

### 7.1. Video Oluşturma Stratejisi
    - **n8n ile Doğrudan Video Oluşturma:** n8n'de karmaşık video düzenleme işlemleri için hazır bir nod bulunmayabilir. Ancak, "Execute Command" nodu ile `ffmpeg` gibi bir komut satırı aracını tetikleyebilirsiniz. Bu, `ffmpeg`'in bilgisayarınızda veya Docker container'ınızda kurulu olmasını gerektirir.
    - **Harici Servis Kullanma:** Bannerbear, Shotstack gibi API tabanlı video oluşturma servislerini "HTTP Request" nodu ile kullanabilirsiniz.
    - **moviepy ile Python Scripti:** Eğer n8n kurulumunuzda "Execute Command" nodu ile Python scriptlerini çalıştırabiliyorsanız (veya n8n'in Code nodu Python destekliyorsa - genelde JS destekler), `moviepy` kullanan bir script yazabilirsiniz.

### 7.2. `ffmpeg` ile Basit Slayt Video (Execute Command Nodu)
    - `ffmpeg` kurulu olmalıdır.
    - Her konuşma metnini bir resim dosyasına (örneğin, metni bir arka plan üzerine yazdırarak) dönüştürmeniz gerekir. Bunu da `ffmpeg` veya ImageMagick (`convert` komutu) ile yapabilirsiniz.
    - Sonra bu resimleri sıralı bir şekilde birleştirip ses dosyalarını ekleyerek video oluşturabilirsiniz.
    - **Örnek `ffmpeg` komutları (çok basitleştirilmiş):**
        - Metni resme çevirme (her slayt için): `ffmpeg -f lavfi -i color=c=blue:s=1280x720:d=5 -vf "drawtext=text='Münazara Metni Burada':fontcolor=white:fontsize=40:x=(w-text_w)/2:y=(h-text_h)/2" slide1.png`
        - Resimleri ve sesi videoya çevirme: `ffmpeg -framerate 1/5 -i slide%d.png -i audio.mp3 -c:v libx264 -c:a aac -pix_fmt yuv420p -shortest output.mp4` (slide1.png, slide2.png... ve birleştirilmiş bir audio.mp3 olduğunu varsayar)
    - "Execute Command" nodunda bu komutları çalıştırırsınız. Dosya yollarına ve sıralamaya dikkat edin.

## Adım 8: Kazananı Belirlemek İçin Basit Bir Kural Sistemi Eklenmesi

### 8.1. Kural Tanımlama
    - **Argüman Sayısı:** En çok argüman üreten LLM. (Her LLM'in argümanlarını saymak için bir değişken tutabilirsiniz).
    - **Kelime Sayısı:** Daha fazla veya daha dengeli kelime kullanan.
    - **Kullanıcı Oyu:** (n8n dışı bir sistemle veya manuel olarak)
    - **Basit Anahtar Kelime Analizi:** Belirli olumlu/olumsuz anahtar kelimelerin sayısına bakılabilir (çok temel).

### 8.2. n8n'de Uygulama
    - **IF Node:** Tanımladığınız kurallara göre kazananı belirlemek için IF nodları kullanabilirsiniz.
        - Örneğin, `llm1_arguman_sayisi > llm2_arguman_sayisi` ise LLM 1 kazanır.
    - **Set Node:** Kazananı bir değişkene atayın.

## Adım 9: Sonuçların Kaydedilmesi ve Çıktıların Sunulması

### 9.1. Veri Kaydetme
    - **Write Binary File:** Ses dosyalarını, videoları kaydetmek için.
    - **Write File:** Metin tabanlı sonuçları (tüm konuşma dökümü, kazanan bilgisi) `.txt` veya `.md` dosyası olarak kaydetmek için.
    - **Spreadsheet File Node / Google Sheets Node / Airtable Node vb.:** Sonuçları bir tabloya yazmak için.
    - **Email Node:** Sonuçları e-posta ile göndermek için.

### 9.2. Çıktı Formatı
    - Tüm konuşmaların metin dökümü.
    - Her LLM için ayrı ses dosyaları veya birleştirilmiş tek bir ses dosyası.
    - (Opsiyonel) Oluşturulan video.
    - Kazanan bilgisi.

## İpuçları ve En İyi Pratikler

- **Credentials (Kimlik Bilgileri):** API anahtarlarınızı doğrudan nodların içine yazmak yerine n8n'in "Credentials" bölümünde saklayın. HTTP Request nodunda "Authentication" > "Generic Credential" seçip oluşturduğunuz kimlik bilgisini seçin.
- **Error Handling (Hata Yönetimi):** "Error Trigger" nodunu kullanarak akışınızda bir hata oluştuğunda (örn: API yanıt vermedi) ne olacağını belirleyebilirsiniz. Örneğin, bir bildirim gönderebilirsiniz.
- **Expression Editor:** n8n'deki Expression Editor (`{{ }}` içine yazılan kodlar) çok güçlüdür. Önceki nodlardan gelen verilere erişmek ve bunları işlemek için kullanılır. Sık sık kullanacaksınız.
- **Execution Log (Çalıştırma Günlüğü):** Akışı çalıştırdıktan sonra her nodun giriş ve çıkış verilerini "Execution Log" bölümünden inceleyebilirsiniz. Bu, hata ayıklama için çok önemlidir.
- **Save Workflow Regularly:** Çalışmalarınızı sık sık kaydedin.
- **Start Small:** Önce küçük parçaları çalışır hale getirin, sonra birleştirin. Örneğin, önce sadece bir LLM'den yanıt almayı başarın, sonra ikincisini ekleyin.

Bu yol haritası sana projen boyunca rehberlik edecektir, Aliqo. Her adımda takıldığın bir yer olursa veya daha fazla detaya ihtiyacın olursa çekinme, sor! Şimdi bu yol haritasını `roadmap.md` dosyasına kaydedelim. 