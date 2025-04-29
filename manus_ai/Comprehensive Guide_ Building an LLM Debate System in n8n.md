# Kapsamlı Rehber: n8n ile LLM Münazara Sistemi Kurma

## 1. Giriş

Bu rehber, n8n otomasyon platformunu kullanarak otomatik bir münazara sistemi oluşturmak için kapsamlı bir kılavuz sunmaktadır. Sistem, kullanıcı tarafından sağlanan bir konu üzerinde, başlangıçta Google'ın Gemma 2.0 Flash modeli için yapılandırılmış iki Büyük Dil Modeli (LLM) arasında yapılandırılmış bir münazarayı yönetir. Her LLM'e farklı bir kişilik atanır ve birkaç tur boyunca kendi pozisyonunu savunur. Üretilen metin yanıtları daha sonra ElevenLabs API kullanılarak konuşmaya dönüştürülür ve her LLM için farklı sesler atanır. Rehber, kurulumu, iş akışı tasarımını, ayrıntılı uygulama adımlarını ve kazanan belirleme ve video oluşturma gibi isteğe bağlı geliştirmeleri kapsar.

Bu sistem, n8n'in görsel iş akışı oluşturucusundan ve HTTP istekleri aracılığıyla harici API'lerle entegre olma yeteneğinden yararlanır.

## 2. Ön Koşullar

İş akışını oluşturmaya başlamadan önce aşağıdakilere sahip olduğunuzdan emin olun:

*   **n8n Örneği:** Çalışan bir n8n örneğine erişim. Bu, n8n Cloud sürümü veya kendi kendine barındırılan (self-hosted) bir örnek olabilir.
*   **API Anahtarları:**
    *   **Google AI Studio API Anahtarı:** Gemini API'sini (Gemma 2.0 Flash dahil) kullanmak için Google AI Studio'dan ([https://aistudio.google.com/](https://aistudio.google.com/)) bir API anahtarı edinin.
    *   **ElevenLabs API Anahtarı:** Metinden konuşmaya dönüştürme için ElevenLabs'tan ([https://elevenlabs.io/](https://elevenlabs.io/)) bir API anahtarı edinin. Ayrıca kullanmak istediğiniz sesler için Ses Kimliklerine (Voice ID) ihtiyacınız olacaktır.
*   **Temel n8n Bilgisi:** n8n arayüzüne aşinalık, düğüm ekleme, bunları bağlama ve temel ifade (expression) kullanımı önerilir.
*   **(İsteğe Bağlı) `ffmpeg`:** Oluşturulan ses dosyalarını birleştirmeyi planlıyorsanız, `ffmpeg`'in n8n yürütme ortamınızda kurulu ve erişilebilir olduğundan emin olun (özellikle kendi kendine barındırılan örnekler için geçerlidir).
*   **(İsteğe Bağlı) Python Ortamı ve Kütüphaneler:** İsteğe bağlı video oluşturma adımını uygulamayı planlıyorsanız, n8n tarafından erişilebilen (örneğin, Execute Command düğümü aracılığıyla) ve `moviepy` ve `Pillow` gibi kütüphanelerin kurulu olduğu bir Python ortamına ihtiyacınız olacaktır.

## 3. n8n Kimlik Bilgisi Kurulumu

API anahtarlarınızı n8n'in kimlik bilgisi yöneticisinde güvenli bir şekilde saklayın:

1.  **Google AI Studio (Gemini API):**
    *   n8n ayarlarınızda **Credentials** (Kimlik Bilgileri) bölümüne gidin.
    *   **Add Credential** (Kimlik Bilgisi Ekle) seçeneğine tıklayın.
    *   Kimlik bilgisi türü olarak **Google API**'yi arayın ve seçin.
    *   **API Key** (API Anahtarı) alanına Google AI Studio API Anahtarınızı yapıştırın.
    *   Kimlik bilgisine tanınabilir bir **Credential Name** (Kimlik Bilgisi Adı) verin (örneğin, `GoogleAiStudioApi`).
    *   **Save** (Kaydet) seçeneğine tıklayın.

2.  **ElevenLabs API:**
    *   n8n ayarlarınızda **Credentials** (Kimlik Bilgileri) bölümüne gidin.
    *   **Add Credential** (Kimlik Bilgisi Ekle) seçeneğine tıklayın.
    *   Kimlik bilgisi türü olarak **Header Auth**'u arayın ve seçin.
    *   **Name** (Ad) alanını `xi-api-key` olarak ayarlayın.
    *   **Value** (Değer) alanını ElevenLabs API Anahtarınız olarak ayarlayın.
    *   Kimlik bilgisine tanınabilir bir **Credential Name** (Kimlik Bilgisi Adı) verin (örneğin, `ElevenLabsApi`).
    *   **Save** (Kaydet) seçeneğine tıklayın.

## 4. İş Akışı Tasarımına Genel Bakış

İş akışı aşağıdaki sırayla çalışır:

1.  **Tetikleyici (Trigger):** Münazara konusuyla manuel olarak (veya webhook aracılığıyla) başlar.
2.  **Başlatma (Initialization):** Kişilikler, ses kimlikleri, tur limitleri ve transkript/ses için boş diziler gibi başlangıç değişkenlerini ayarlar.
3.  **Münazara Döngüsü (Debate Loop):** Maksimum tur limitine göre sıra alma sürecini kontrol eder.
4.  **Konuşmacı/Prompt Hazırlama:** Sıranın kimde olduğunu (LLM1 veya LLM2) belirler ve önceki yanıtı içeren uygun prompt'u oluşturur.
5.  **LLM API Çağrısı:** Prompt'u bir HTTP İsteği aracılığıyla Gemma 2.0 Flash API'sine gönderir.
6.  **Durum Güncelleme (State Update):** LLM'in metin yanıtını transkripte kaydeder ve `last_response` değişkenini günceller.
7.  **TTS API Çağrısı:** Konuşma oluşturmak için metin yanıtını bir HTTP İsteği aracılığıyla ElevenLabs API'sine gönderir.
8.  **Ses Depolama (Audio Storage):** Oluşturulan ses dosyası hakkındaki meta verileri saklar.
9.  **Turu Artır (Increment Turn):** Tur sayacını artırır.
10. **Döngüye Geri Dön (Loop Back):** Döngü koşulu kontrolüne geri döner.
11. **İşlem Sonrası (Post-Processing - İsteğe Bağlı):** Döngü bittikten sonra kazanan belirleme, ses birleştirme veya video oluşturma gibi görevleri gerçekleştirir.
12. **Nihai Çıktı (Final Output):** Nihai sonuçları (transkript, ses/video dosyaları, kazanan ilanı) hazırlar ve sunar.

*(İş akışı diyagramı veya görsel temsil ideal olarak nihai çıktı formatında mümkünse buraya eklenmelidir)*

## 5. Ayrıntılı İş Akışı Uygulaması

İş akışını n8n tuvalinizde oluşturmak için şu adımları izleyin:

### Adım 5.1: İş Akışı Tetikleyicisi (Manuel)

*   Tuvalinize bir **Manual** (Manuel) tetikleyici düğümü ekleyin. Bu genellikle varsayılan başlangıç düğümüdür.
*   **Yapılandırma:**
    *   Parametrelerini açmak için düğüme tıklayın.
    *   **Inputs** (Girdiler) altında, **Add Input** (Girdi Ekle) seçeneğine tıklayın.
    *   **String** (Metin) seçin.
    *   **Name** (Ad) alanını `debate_topic` olarak ayarlayın.
    *   **Description** (Açıklama) alanını `Münazara konusunu girin.` olarak ayarlayın.
    *   **Required** (Gerekli) kutusunu işaretleyin.

### Adım 5.2: Başlatma (Set Düğümü)

*   Bir **Set** düğümü ekleyin ve **Manual** tetikleyici düğümünü girişine bağlayın.
*   Düğümü `Initialize Debate` (Münazarayı Başlat) olarak yeniden adlandırın (isteğe bağlı, ancak iyi bir uygulama).
*   **Yapılandırma:**
    *   **Keep Only Set** (Sadece Ayarlananları Tut) seçeneğini **True** (Doğru) olarak ayarlayın.
    *   Aşağıdakileri eklemek için **Add Value** (Değer Ekle) seçeneğine birden çok kez tıklayın:
        *   **Name:** `debate_topic`, **Value:** `{{ $json.debate_topic }}` (Mod: Expression, Tür: String)
        *   **Name:** `llm1_persona`, **Value:** `"Karbon emisyonlarına daha sıkı düzenlemeler getirilmesini savunan dikkatli bir çevre bilimcisisiniz."` (Mod: Fixed, Tür: String - *Bu kişiliği özelleştirin*)
        *   **Name:** `llm2_persona`, **Value:** `"İklim değişikliğini çözmenin anahtarının düzenleme değil, teknolojik yenilik olduğunu savunan iyimser bir ekonomistsiniz."` (Mod: Fixed, Tür: String - *Bu kişiliği özelleştirin*)
        *   **Name:** `llm1_voice_id`, **Value:** `SENIN_ELEVENLABS_SES_KIMLIGIN_1` (Mod: Fixed, Tür: String - *Gerçek Ses Kimliği ile değiştirin*)
        *   **Name:** `llm2_voice_id`, **Value:** `SENIN_ELEVENLABS_SES_KIMLIGIN_2` (Mod: Fixed, Tür: String - *Gerçek Ses Kimliği ile değiştirin*)
        *   **Name:** `max_turns`, **Value:** `6` (Mod: Fixed, Tür: Number - *Gerektiğinde ayarlayın*)
        *   **Name:** `current_turn`, **Value:** `0` (Mod: Fixed, Tür: Number)
        *   **Name:** `debate_transcript`, **Value:** `[]` (Mod: Expression, Tür: JSON)
        *   **Name:** `debate_audio_files`, **Value:** `[]` (Mod: Expression, Tür: JSON)
        *   **Name:** `last_response`, **Value:** `""` (Mod: Fixed, Tür: String)

### Adım 5.3: Münazara Döngüsü Koşulu (IF Düğümü)

*   Bir **IF** düğümü ekleyin ve `Initialize Debate` düğümünü girişine bağlayın.
*   Düğümü `Check Turn Limit` (Tur Limitini Kontrol Et) olarak yeniden adlandırın.
*   **Yapılandırma:**
    *   **Add Condition** (Koşul Ekle) -> **Number** (Sayı) seçeneğine tıklayın.
    *   **Value 1** (Değer 1) alanını Expression: `{{ $json.current_turn }}` olarak ayarlayın.
    *   **Operation** (İşlem) alanını `Smaller` (Küçüktür) olarak ayarlayın.
    *   **Value 2** (Değer 2) alanını Expression: `{{ $json.max_turns }}` olarak ayarlayın.

### Adım 5.4: Mevcut Konuşmacıyı Belirleme ve Prompt Hazırlama (IF/Set Düğümleri)

*   Başka bir **IF** düğümü ekleyin ve `Check Turn Limit` düğümünün **true** (doğru) çıkışını girişine bağlayın.
*   Düğümü `Is LLM1 Turn?` (Sıra LLM1'de mi?) olarak yeniden adlandırın.
*   **Yapılandırma:**
    *   **Add Condition** (Koşul Ekle) -> **Number** (Sayı) seçeneğine tıklayın.
    *   **Value 1** (Değer 1) alanını Expression: `{{ $json.current_turn % 2 }}` olarak ayarlayın.
    *   **Operation** (İşlem) alanını `Equal` (Eşittir) olarak ayarlayın.
    *   **Value 2** (Değer 2) alanını Fixed: `0` olarak ayarlayın.

*   **LLM 1 Dalı (True Çıkışı):**
    *   Bir **Set** düğümü ekleyin ve `Is LLM1 Turn?` düğümünün **true** çıkışını girişine bağlayın.
    *   Düğümü `Prepare LLM1 Prompt` (LLM1 Prompt'unu Hazırla) olarak yeniden adlandırın.
    *   **Yapılandırma:**
        *   **Keep Only Set** (Sadece Ayarlananları Tut) seçeneğini **False** (Yanlış) olarak ayarlayın.
        *   Değer Ekle: **Name:** `current_voice_id`, **Value:** `{{ $json.llm1_voice_id }}` (Mod: Expression, Tür: String).
        *   Değer Ekle: **Name:** `current_prompt`, **Value:** (Mod: Expression, Tür: String)
            ```javascript
            // Münazara konusunu, LLM'in kişiliğini, son yanıtı ve mevcut turu al
            const topic = $json.debate_topic;
            const persona = $json.llm1_persona;
            const lastResponse = $json.last_response;
            const turn = $json.current_turn;
            
            // Eğer ilk tur ise (tur 0), açılış argümanı prompt'u oluştur
            if (turn === 0) {
              return `${persona}. Münazara konusu: '${topic}'. Lütfen açılış argümanınızı sunun.`;
            } 
            // Değilse, rakibin son yanıtını içeren bir yanıt prompt'u oluştur
            else {
              return `${persona}. Münazara konusu: '${topic}'. Rakibiniz az önce şunu söyledi: '${lastResponse}'. Lütfen onun argümanına yanıt verin ve karşı argümanınızı sunun. Yanıtınızı kısa tutun.`;
            }
            ```

*   **LLM 2 Dalı (False Çıkışı):**
    *   Bir **Set** düğümü ekleyin ve `Is LLM1 Turn?` düğümünün **false** çıkışını girişine bağlayın.
    *   Düğümü `Prepare LLM2 Prompt` (LLM2 Prompt'unu Hazırla) olarak yeniden adlandırın.
    *   **Yapılandırma:**
        *   **Keep Only Set** (Sadece Ayarlananları Tut) seçeneğini **False** (Yanlış) olarak ayarlayın.
        *   Değer Ekle: **Name:** `current_voice_id`, **Value:** `{{ $json.llm2_voice_id }}` (Mod: Expression, Tür: String).
        *   Değer Ekle: **Name:** `current_prompt`, **Value:** (Mod: Expression, Tür: String)
            ```javascript
            // Münazara konusunu, LLM'in kişiliğini, son yanıtı ve mevcut turu al
            const topic = $json.debate_topic;
            const persona = $json.llm2_persona;
            const lastResponse = $json.last_response;
            const turn = $json.current_turn;
            
            // Eğer ikinci tur ise (tur 1), rakibin ilk argümanına yanıt veren açılış prompt'u oluştur
            if (turn === 1) {
              return `${persona}. Münazara konusu: '${topic}'. Rakibiniz şununla başladı: '${lastResponse}'. Lütfen açılış argümanınızı sunun, gerekirse onun puanlarına yanıt verin.`;
            } 
            // Değilse, rakibin son yanıtını içeren bir yanıt prompt'u oluştur
            else {
              return `${persona}. Münazara konusu: '${topic}'. Rakibiniz az önce şunu söyledi: '${lastResponse}'. Lütfen onun argümanına yanıt verin ve karşı argümanınızı sunun. Yanıtınızı kısa tutun.`;
            }
            ```

*   Bir **Merge** (Birleştir) düğümü ekleyin. Hem `Prepare LLM1 Prompt` hem de `Prepare LLM2 Prompt` düğümlerinin çıkışlarını girişlerine bağlayın. **Mode** (Mod) seçeneğini `Append` (Ekle) olarak ayarlayın.

### Adım 5.5: LLM API Çağrısı (HTTP Request Düğümü)

*   Bir **HTTP Request** (HTTP İsteği) düğümü ekleyin ve **Merge** düğümü çıkışını girişine bağlayın.
*   Düğümü `Call Gemma API` (Gemma API'sini Çağır) olarak yeniden adlandırın.
*   **Yapılandırma:**
    *   **Authentication** (Kimlik Doğrulama): `Predefined Credential` (Önceden Tanımlanmış Kimlik Bilgisi).
    *   **Credential** (Kimlik Bilgisi): `GoogleAiStudioApi` kimlik bilginizi seçin.
    *   **Request Method** (İstek Yöntemi): `POST`.
    *   **URL:** İfade kullanın: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key={{ $credentials.GoogleAiStudioApi.apiKey }}`.
    *   **Send Body** (Gövde Gönder): `On` (Açık).
    *   **Body Content Type** (Gövde İçerik Türü): `JSON`.
    *   **Body** (Gövde): İfade kullanın:
        ```json
        {
          // İstek içeriğini belirtir
          "contents": [
            {
              // İçeriğin bölümlerini belirtir
              "parts": [
                {
                  // Gönderilecek metin (prompt)
                  "text": "{{ $json.current_prompt }}" 
                }
              ]
            }
          ]
        }
        ```
    *   **Options** (Seçenekler) -> **Response** (Yanıt) bölümünü genişletin.
    *   **Response Format** (Yanıt Formatı) seçeneğini `JSON` olarak ayarlayın.

### Adım 5.6: Transkripti ve Son Yanıtı Güncelle (Set Düğümü)

*   Bir **Set** düğümü ekleyin ve `Call Gemma API` düğümü çıkışını girişine bağlayın.
*   Düğümü `Update State` (Durumu Güncelle) olarak yeniden adlandırın.
*   **Yapılandırma:**
    *   **Keep Only Set** (Sadece Ayarlananları Tut) seçeneğini **False** (Yanlış) olarak ayarlayın.
    *   Değer Ekle: **Name:** `current_response`, **Value:** `{{ $json.candidates[0].content.parts[0].text }}` (Mod: Expression, Tür: String - *Gerçek API yanıtındaki yolu doğrulayın: API yanıtından gelen adayın ilk içeriğinin ilk bölümündeki metni alır*).
    *   Değer Ekle: **Name:** `last_response`, **Value:** `{{ $json.candidates[0].content.parts[0].text }}` (Mod: Expression, Tür: String - *Yolu doğrulayın: Bir sonraki turda kullanılmak üzere mevcut yanıtı saklar*).
    *   Değer Ekle: **Name:** `debate_transcript`, **Value:** (Mod: Expression, Tür: JSON)
        ```javascript
        // Mevcut transkripti al
        const transcript = $json.debate_transcript;
        // Mevcut turun çift mi tek mi olduğuna bakarak konuşmacıyı belirle (LLM1 veya LLM2)
        const speaker = ($json.current_turn % 2 === 0) ? 'LLM1' : 'LLM2';
        // API yanıtından gelen metni al (yolu gerekirse ayarlayın)
        const text = $json.candidates[0].content.parts[0].text; 
        // Konuşmacı ve metni içeren yeni bir nesneyi transkripte ekle
        transcript.push({ speaker: speaker, text: text });
        // Güncellenmiş transkripti döndür
        return transcript;
        ```

### Adım 5.7: ElevenLabs TTS Çağrısı (HTTP Request Düğümü)

*   Bir **HTTP Request** (HTTP İsteği) düğümü ekleyin ve `Update State` düğümü çıkışını girişine bağlayın.
*   Düğümü `Generate Speech` (Konuşma Oluştur) olarak yeniden adlandırın.
*   **Yapılandırma:**
    *   **Authentication** (Kimlik Doğrulama): `Predefined Credential` (Önceden Tanımlanmış Kimlik Bilgisi).
    *   **Credential** (Kimlik Bilgisi): `ElevenLabsApi` kimlik bilginizi seçin.
    *   **Request Method** (İstek Yöntemi): `POST`.
    *   **URL:** İfade kullanın: `https://api.elevenlabs.io/v1/text-to-speech/{{ $json.current_voice_id }}`.
    *   **Send Body** (Gövde Gönder): `On` (Açık).
    *   **Body Content Type** (Gövde İçerik Türü): `JSON`.
    *   **Body** (Gövde): İfade kullanın:
        ```json
        {
          // Konuşmaya dönüştürülecek metin
          "text": "{{ $json.current_response }}",
          // Kullanılacak ses modeli (isteğe bağlı, gerekirse değiştirin)
          "model_id": "eleven_multilingual_v2" 
        }
        ```
        *İstenirse/gerekirse `model_id`'yi ayarlayın.*
    *   **Options** (Seçenekler) -> **Response** (Yanıt) bölümünü genişletin.
    *   **Response Format** (Yanıt Formatı) seçeneğini `File` (Dosya) olarak ayarlayın.
    *   **Binary Property** (İkili Özellik) seçeneğini `data` olarak ayarlayın.

### Adım 5.8: Ses Verisini Depola (Set Düğümü - İsteğe Bağlı)

*   *Bu adım öncelikle meta verileri saklar; ikili ses n8n öğesine eklenir.* Bir **Set** düğümü ekleyin ve `Generate Speech` düğümü çıkışını girişine bağlayın.
*   Düğümü `Store Audio Meta` (Ses Meta Verisini Depola) olarak yeniden adlandırın.
*   **Yapılandırma:**
    *   **Keep Only Set** (Sadece Ayarlananları Tut) seçeneğini **False** (Yanlış) olarak ayarlayın.
    *   Değer Ekle: **Name:** `debate_audio_files`, **Value:** (Mod: Expression, Tür: JSON)
        ```javascript
        // Mevcut ses dosyaları dizisini al
        const audioFiles = $json.debate_audio_files;
        // Mevcut turun çift mi tek mi olduğuna bakarak konuşmacıyı belirle
        const speaker = ($json.current_turn % 2 === 0) ? 'LLM1' : 'LLM2';
        // Ses dosyası için bir isim oluştur (örn: turn_0_LLM1.mp3)
        const filename = `turn_${$json.current_turn}_${speaker}.mp3`; 
        // Meta veriyi sakla. Gerçek ikili veri (ses dosyası) $binary.data içindedir
        audioFiles.push({ speaker: speaker, filename: filename, binaryPropertyName: 'data' }); 
        // Güncellenmiş ses dosyaları dizisini döndür
        return audioFiles;
        ```

### Adım 5.9: Tur Sayacını Artır (Set Düğümü)

*   Bir **Set** düğümü ekleyin ve `Store Audio Meta` düğümü çıkışını girişine bağlayın.
*   Düğümü `Increment Turn` (Turu Artır) olarak yeniden adlandırın.
*   **Yapılandırma:**
    *   **Keep Only Set** (Sadece Ayarlananları Tut) seçeneğini **False** (Yanlış) olarak ayarlayın.
    *   Değer Ekle: **Name:** `current_turn`, **Value:** `{{ $json.current_turn + 1 }}` (Mod: Expression, Tür: Number).

### Adım 5.10: Döngüye Geri Dön

*   `Increment Turn` düğümünün çıkışını `Check Turn Limit` **IF** düğümünün (Adım 5.3) girişine geri bağlayın.

### Adım 5.11: Döngü Sonrası İşlem ve Nihai Çıktı

*   `Check Turn Limit` **IF** düğümünün (Adım 5.3) **false** (yanlış) çıkışını nihai işlem dizinizin başlangıcına bağlayın.
*   Bir **Set** düğümü ekleyin.
*   `Prepare Final Output` (Nihai Çıktıyı Hazırla) olarak yeniden adlandırın.
*   **Yapılandırma:**
    *   **Keep Only Set** (Sadece Ayarlananları Tut) seçeneğini **True** (Doğru) olarak ayarlayın.
    *   Değer Ekle: **Name:** `final_transcript`, **Value:** `{{ $json.debate_transcript }}` (Mod: Expression, Tür: JSON).
    *   Değer Ekle: **Name:** `audio_metadata`, **Value:** `{{ $json.debate_audio_files }}` (Mod: Expression, Tür: JSON).
    *   *(Kazanan belirleme veya birleştirilmiş ses/video yolları gibi isteğe bağlı adımlar uygularsanız buraya başka değerler ekleyin)*.
*   Bu son **Set** düğümünün yürütme sonucu, münazara transkriptini ve oluşturulan ses dosyaları hakkındaki meta verileri içerecektir. Bir Webhook tetikleyicisi kullanıyorsanız, bu Set düğümünden sonra bir **Respond to Webhook** (Webhook'a Yanıt Ver) düğümü ekleyin.

## 6. İsteğe Bağlı Geliştirmeler

*   **Kazanan Belirleme:**
    *   Döngü bittikten sonra (`Check Turn Limit` false çıkışından bağlanarak) bir **Code** (Kod) düğümü ekleyin.
    *   `debate_transcript` dizisini analiz etmek için JavaScript kodu yazın. Basit mantık anahtar kelimeleri sayabilir veya niteliksel bir değerlendirme için daha karmaşık bir yaklaşım (potansiyel olarak HTTP İsteği aracılığıyla başka bir LLM çağrısı) kullanabilirsiniz.
    *   Sonucu bir `winner_declaration` (kazanan ilanı) metni olarak çıktılayın.
    *   `winner_declaration`'ı `Prepare Final Output` düğümüne dahil edin.
*   **Ses Dosyalarını Birleştirme:**
    *   *Ortamda `ffmpeg` gerektirir.*
    *   Döngüden sonra, her bir ses ikili verisini (`debate_audio_files` içinde başvurulan) geçici bir dosyaya yazmak için düğümler ekleyin (örneğin, **Write Binary File** düğümünü kullanarak).
    *   Bu geçici dosyaları listeleyen bir metin dosyası (`filelist.txt`) oluşturun (`file 'temp_audio_0.mp3'`, vb.).
    *   `ffmpeg -f concat -safe 0 -i filelist.txt -c copy C:\n8n_data\combined_debate.mp3` komutuyla bir **Execute Command** (Komut Çalıştır) düğümü kullanın. (Windows için örnek yol, kendi yolunuzu ayarlayın).
    *   `C:\n8n_data\combined_debate.mp3` yolunu `Prepare Final Output` düğümüne dahil edin.
    *   Geçici dosyaları temizlemek için düğümler ekleyin (örn. `del C:\n8n_data\temp_audio_*.mp3`).
*   **Video Oluşturma:**
    *   *Python, `moviepy`, `Pillow` vb. gerektirir.*
    *   Transkripti ve ses dosyası yollarını kabul eden bir Python betiği (`video_script.py`) geliştirin.
    *   Betik, her metin segmenti için görüntüler/slaytlar oluşturmalı ve bunları `moviepy` kullanarak ilgili sesle birleştirmelidir.
    *   Döngüden sonra (ve potansiyel olarak sesi birleştirdikten sonra), bir **Execute Command** düğümü kullanın: `python C:\path\to\video_script.py --transcript "{{ JSON.stringify($json.debate_transcript) }}" --audio C:\n8n_data\combined_debate.mp3 --output C:\n8n_data\debate_video.mp4` (Betik için argümanları ve yolları gerektiği gibi ayarlayın).
    *   `C:\n8n_data\debate_video.mp4` yolunu `Prepare Final Output` düğümüne dahil edin.

## 7. Test Etme ve Hata Ayıklama

*   **Artımlı Test Edin:** Her yeni düğüm veya karmaşık ifade ekledikten sonra iş akışını adım adım yürütün.
*   **Girdi/Çıktıyı İnceleyin:** Verilerin doğru aktığından emin olmak için yürütme sırasında her düğümün girdi ve çıktı verilerini kontrol edin.
*   **API Yanıtları:** Gemma ve ElevenLabs API'lerinden gelen yanıtların yapısını doğrulayın ve ifadeleri (örneğin `{{ $json.candidates[0].content.parts[0].text }}`) buna göre ayarlayın.
*   **Hata İşleme:** API hatalarını veya beklenmeyen sorunları yakalamak için **Error Trigger** (Hata Tetikleyici) düğümlerini geri dönüş yollarına veya bildirim düğümlerine (örneğin, Slack/E-posta) bağlamayı düşünün.
*   **İfade Düzenleyici:** JavaScript ifadelerinizi test etmek için n8n ifade düzenleyicisinin önizleme özelliğini kullanın.

## 8. Sonuç

Bu rehber, n8n'de bir LLM münazara sistemi kurmak için sağlam bir çerçeve sunmaktadır. Bu adımları izleyerek, Gemma 2.0 Flash gibi LLM'lerin ve ElevenLabs gibi metinden konuşmaya hizmetlerinin gücünden yararlanarak ilgi çekici münazaralar üreten bir iş akışı oluşturabilirsiniz. Kişilikleri, ses kimliklerini ve isteğe bağlı geliştirmeleri özel ihtiyaçlarınıza uyacak şekilde özelleştirmeyi unutmayın. Mutlu otomasyonlar!

