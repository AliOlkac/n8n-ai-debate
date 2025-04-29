## n8n İş Akışı Tasarımı: LLM Münazara Sistemi

Bu belge, iki Büyük Dil Modeli (LLM), özellikle Gemma 2.0 Flash arasında bir münazarayı yöneten ve ElevenLabs kullanarak metinden konuşmaya dönüştürme yapan n8n iş akışının önerilen yapısını özetlemektedir.

**Temel Bileşenler ve Mantık:**

1.  **İş Akışı Tetikleyicisi (Manuel/Webhook):**
    *   **Düğüm:** Manuel Tetikleyici veya Webhook düğümü.
    *   **Amaç:** Münazara sürecini başlatmak.
    *   **Girdi:** Kullanıcıdan münazara konusunu kabul eder.
    *   **Çıktı:** Münazara konusunu bir sonraki düğüme iletir.

2.  **Başlatma (Set Düğümü):**
    *   **Düğüm:** Set düğümü.
    *   **Amaç:** Münazara için başlangıç parametrelerini ve durum değişkenlerini tanımlamak.
    *   **Yapılandırma:**
        *   Girdi `debate_topic` değerini sakla.
        *   `llm1_persona` ve `llm1_initial_prompt` tanımla (konu ve kişiliği içeren).
        *   `llm2_persona` ve `llm2_initial_prompt` tanımla (konu ve kişiliği içeren).
        *   ElevenLabs için `llm1_voice_id` ve `llm2_voice_id` tanımla.
        *   `max_turns` ayarla (örneğin, 3 tur, yani toplam 6 konuşma sırası).
        *   `current_turn = 0` olarak başlat.
        *   `debate_transcript = []` olarak başlat (metin yanıtlarını saklamak için dizi).
        *   `debate_audio_files = []` olarak başlat (ses dosyası referanslarını/verilerini saklamak için dizi).
        *   `last_response = ""` olarak başlat.

3.  **Münazara Döngüsü (Loop Over Items / IF Düğümü Mantığı):**
    *   **Düğüm:** Bir Loop Over Items düğümü ( `max_turns * 2` kez dönen) veya iş akışı özyinelemesi/zincirlemesi ile birleştirilmiş IF düğümleri kullanılabilir.
    Daha basit bir yaklaşım, `current_turn < max_turns * 2` kontrolü yapan IF düğümleri olabilir.
    *   **Amaç:** Belirtilen tur sayısı için sıra alma dizisini kontrol etmek.

4.  **Mevcut Konuşmacıyı Belirleme ve Prompt Hazırlama (IF/Set Düğümleri):**
    *   **Düğüm:** `current_turn` değerinin çift (LLM 1) mi yoksa tek (LLM 2) mi olduğunu kontrol eden IF düğümü.
    *   **Düğüm:** IF dalları içindeki Set düğüm(ler)i.
    *   **Amaç:** Doğru LLM'i, kişiliği seçmek ve sıraya göre prompt'u oluşturmak.
    *   **Mantık:**
        *   Eğer `current_turn == 0`, `llm1_initial_prompt` kullan.
        *   Eğer `current_turn == 1`, `last_response` (LLM 1'den) içeren `llm2_initial_prompt` kullan.
        *   Eğer `current_turn > 1` ve çift (LLM 1), `llm1_persona`, `debate_topic` ve `last_response` (LLM 2'den) kullanarak prompt oluştur.
        *   Eğer `current_turn > 1` ve tek (LLM 2), `llm2_persona`, `debate_topic` ve `last_response` (LLM 1'den) kullanarak prompt oluştur.
    *   **Çıktı:** `current_prompt`, `current_voice_id`.

5.  **LLM API Çağrısı (HTTP Request Düğümü):**
    *   **Düğüm:** HTTP Request düğümü.
    *   **Amaç:** `current_prompt`'u Gemma 2.0 Flash API'sine göndermek.
    *   **Yapılandırma:**
        *   **Method:** POST
        *   **URL:** `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key={{ $credentials.GoogleAiStudioApi.apiKey }}` (API anahtarı için n8n kimlik bilgilerini kullanarak).
        *   **Body (JSON):** `{"contents": [{"parts":[{"text": "{{ $json.current_prompt }}"}]}]}`
        *   **Authentication:** Önceden yapılandırılmış Google AI Studio kimlik bilgilerini kullan.
    *   **Çıktı:** LLM yanıt metni (örneğin, `$json.candidates[0].content.parts[0].text`). Bunu `current_response` gibi bir değişkende sakla.

6.  **Transkripti ve Son Yanıtı Güncelle (Set Düğümü):**
    *   **Düğüm:** Set düğümü.
    *   **Amaç:** Mevcut yanıtı saklamak ve genel transkripti güncellemek.
    *   **Yapılandırma:**
        *   `last_response = current_response` ayarla.
        *   `debate_transcript` dizisine `{ speaker: (current_turn % 2 == 0 ? 'LLM1' : 'LLM2'), text: current_response }` ekle.

7.  **ElevenLabs TTS Çağrısı (HTTP Request Düğümü):**
    *   **Düğüm:** HTTP Request düğümü.
    *   **Amaç:** `current_response` metnini `current_voice_id` kullanarak konuşmaya dönüştürmek.
    *   **Yapılandırma:**
        *   **Method:** POST
        *   **URL:** `https://api.elevenlabs.io/v1/text-to-speech/{{ $json.current_voice_id }}`
        *   **Headers:** Önceden yapılandırılmış ElevenLabs kimlik bilgilerini (`xi-api-key`) kullan.
        *   **Body (JSON):** `{"text": "{{ $json.current_response }}", "model_id": "eleven_multilingual_v2"}` (Gerekirse modeli belirt).
        *   **Response Format:** File.
    *   **Çıktı:** Ses verisi. Referans/ikili veriyi `debate_audio_files` dizisinde sakla (örneğin, `{ speaker: (current_turn % 2 == 0 ? 'LLM1' : 'LLM2'), audio_data: $binary.data }`).

8.  **Tur Sayacını Artır (Set Düğümü):**
    *   **Düğüm:** Set düğümü.
    *   **Amaç:** Bir sonraki tura geçmek.
    *   **Yapılandırma:** `current_turn = current_turn + 1`.

9.  **Döngü Devamı/Sonu (Adım 3/IF Düğümüne Geri Bağla):**
    *   Adım 8'in çıkışını Adım 3'teki döngü/IF koşulunun girişine geri bağla.

10. **Döngü Sonrası İşlem (Döngü Bittikten Sonra):**
    *   **(İsteğe Bağlı) Kazanan Belirleme (Code/AI Düğümü):**
        *   **Düğüm:** Code düğümü (basit kurallar) veya başka bir AI Model düğümü (örneğin, Basic LLM Chain).
        *   **Amaç:** Kazananı belirlemek için `debate_transcript`'i analiz etmek.
        *   **Girdi:** `debate_transcript`.
        *   **Çıktı:** `winner_declaration` metni.
    *   **(İsteğe Bağlı) Ses Birleştirme (Execute Command/Code Düğümü):**
        *   **Düğüm:** Execute Command düğümü (`ffmpeg` kuruluysa) veya Code düğümü (bir kütüphane kullanılıyorsa).
        *   **Amaç:** `debate_audio_files` içindeki bireysel ses dosyalarını tek bir dosyada birleştirmek.
        *   **Girdi:** `debate_audio_files`.
        *   **Çıktı:** Birleştirilmiş ses dosyasının yolu.
    *   **(İsteğe Bağlı) Video Oluşturma (Execute Command Düğümü):**
        *   **Düğüm:** Execute Command düğümü.
        *   **Amaç:** Metin/sesten bir video oluşturmak için `moviepy` gibi bir araç kullanmak (Python ortamı kurulumu ve betik gerektirir).
        *   **Girdi:** `debate_transcript`, `debate_audio_files` (veya birleştirilmiş ses).
        *   **Çıktı:** Video dosyasının yolu.

11. **Nihai Çıktı (Set/Respond Düğümü):**
    *   **Düğüm:** Nihai çıktı yapısını hazırlamak için Set düğümü.
    *   **Düğüm:** Respond to Webhook düğümü (webhook tetikleyicisi kullanılıyorsa) veya iş akışını basitçe bitir (manuel tetikleyiciyse, sonuçlar son düğüm yürütmesindedir).
    *   **Amaç:** Sonuçları kullanıcıya sunmak.
    *   **Çıktı Verisi:**
        *   `debate_transcript` (tam metin).
        *   Birleştirilmiş ses dosyası yolu/data.
        *   `winner_declaration` (isteğe bağlı).
        *   Video dosyası yolu/data (isteğe bağlı).

**Kimlik Bilgisi İşleme:**
*   Google AI Studio API Anahtarı: n8n'in yerleşik kimlik bilgisi yöneticisinde güvenli bir şekilde sakla.
*   ElevenLabs API Anahtarı: n8n'in yerleşik kimlik bilgisi yöneticisinde (muhtemelen 'Header Auth' türü veya araştırmada açıklandığı gibi özel bir tür olarak) güvenli bir şekilde sakla.

**Hata İşleme:**
*   API çağrıları için temel hata işlemeyi (örneğin, Error Trigger düğümünü kullanarak veya düğüm başarı durumunu kontrol ederek) uygula.
