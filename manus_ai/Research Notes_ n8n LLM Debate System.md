## Araştırma Notları: n8n LLM Münazara Sistemi

### n8n Platform Yetenekleri:
*   Otomasyonları oluşturmak için görsel iş akışı düzenleyici.
*   400'den fazla önceden yapılandırılmış entegrasyonu destekler.
*   Temel düğümler şunları içerir: HTTP Request, Code (JS/Python), Merge, Loop, If, Switch, vb.
*   Özel AI düğümleri (örneğin, OpenAI, Basic LLM Chain) sağlar ve LangChain'i destekler.
*   Kendi kendine barındırılabilir (self-hosted) veya bulut hizmeti aracılığıyla kullanılabilir.
*   HTTP Request düğümü, herhangi bir REST API'ye bağlanmak için çok yönlüdür.
*   API anahtarı yönetimi için özel kimlik bilgilerini destekler.

### Gemma 2.0 Flash (Google AI Studio / Gemini API) Entegrasyonu:
*   n8n'in HTTP Request düğümü aracılığıyla entegrasyon.
*   Bir Google AI Studio API Anahtarı gerektirir.
*   API Uç Noktası (metin oluşturma örneği): `POST https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=SENIN_API_ANAHTARIN`
*   İstek Gövdesi (JSON): `parts` içeren `contents` dizisini içerir (örneğin, `{"text": "Promptunuz buraya"}`).
*   API anahtarını güvenli bir şekilde işlemek gerekir (örneğin, n8n kimlik bilgilerini kullanarak).

### ElevenLabs API Entegrasyonu:
*   n8n'in HTTP Request düğümü aracılığıyla entegrasyon.
*   Bir ElevenLabs API Anahtarı gerektirir.
*   API Anahtarı, n8n'in özel kimlik bilgilerinde bir başlık (header) olarak yapılandırılmalıdır: `{"headers": {"xi-api-key": "SENIN_ELEVENLABS_API_ANAHTARIN"}}`.
*   Metinden Konuşmaya Uç Noktası (örnek): `POST https://api.elevenlabs.io/v1/text-to-speech/{voice_id}`
*   URL yolunda `voice_id` gerektirir.
*   İstek Gövdesi (JSON): Dönüştürülecek `text`'i ve potansiyel olarak `model_id` veya `voice_settings` gibi diğer parametreleri içerir.
*   Yanıt tipik olarak bir ses dosyasıdır (örneğin, MP3).
*   Her LLM münazaracısı için farklı `voice_id` seçmek gerekir.
