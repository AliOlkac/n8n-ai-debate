# n8n ile LLM Münazara Sistemi Proje İçeriği

## 1. Proje Amacı ve Genel Tanım
Bu proje, iki farklı LLM (şimdilik Gemma 2.0 Flash) modelinin bir konu hakkında karşılıklı münazara yapmasını sağlayan, n8n üzerinde otomasyon akışıyla çalışan bir sistemdir. Her LLM'e farklı kişilik ve argüman promptları verilecek, etik kurallar çerçevesinde sırayla karşılıklı konuşacaklar. Sonuçta, konuşmalar ses dosyasına veya slayt şeklinde bir videoya dönüştürülecek.

## 2. Kullanılan Teknolojiler ve Kütüphaneler
- **n8n:** Otomasyon iş akışlarını görsel olarak oluşturmak için kullanılır.
- **Gemma 2.0 Flash (veya başka LLM):** Münazara için kullanılacak yapay zeka modeli.
- **ElevenLabs API:** Metinleri ses dosyasına dönüştürmek için kullanılır.
- **(Opsiyonel) moviepy veya benzeri:** Slayt şeklinde video oluşturmak için kullanılabilir.

## 3. Sistem Akış Diyagramı (Kısa Açıklama)
1. Kullanıcı bir münazara konusu belirler.
2. Her iki LLM'e farklı kişilik ve argüman promptları atanır.
3. LLM'ler sırayla karşılıklı argüman üretir.
4. Üretilen metinler ElevenLabs ile ses dosyasına dönüştürülür.
5. (Opsiyonel) Metinler slayt şeklinde birleştirilip video oluşturulur.
6. Kazanan belirlenir (basit bir kural veya manuel seçimle).

## 4. Yapılacaklar Listesi (To Do List)

1. [ ] n8n kurulumu ve temel akışın oluşturulması
2. [ ] LLM (Gemma 2.0 Flash) API entegrasyonunun yapılması
3. [ ] Promptların ve kişiliklerin tasarlanması
4. [ ] Sıralı münazara akışının n8n'de kurulması
5. [ ] ElevenLabs API ile metinlerin ses dosyasına dönüştürülmesi
6. [ ] (Opsiyonel) Metinlerin slayt şeklinde video haline getirilmesi
7. [ ] Kazananı belirlemek için basit bir kural sistemi eklenmesi
8. [ ] Sonuçların kaydedilmesi ve çıktıların sunulması

## 5. Her Adımın Detaylı Açıklaması

### 1. n8n Kurulumu ve Temel Akış
- n8n'in kurulumu yapılacak ve temel bir workflow oluşturulacak.
- Akışta, kullanıcıdan konu alınacak ve süreç başlatılacak.

### 2. LLM API Entegrasyonu
- Gemma 2.0 Flash API'si n8n'e entegre edilecek.
- Her iki LLM için farklı promptlar hazırlanacak.

### 3. Prompt ve Kişilik Tasarımı
- Her LLM'e farklı kişilik ve argüman promptları atanacak.
- Promptlar etik ve mantıklı tartışma için optimize edilecek.

### 4. Sıralı Münazara Akışı
- LLM'ler sırayla konuşacak şekilde workflow tasarlanacak.
- Her adımda üretilen metin bir sonraki adıma aktarılacak.

### 5. ElevenLabs API ile Seslendirme
- Üretilen metinler ElevenLabs API ile ses dosyasına dönüştürülecek.
- Her LLM'in sesi farklı olacak şekilde ayarlanacak.

### 6. (Opsiyonel) Slayt Şeklinde Video Oluşturma
- Metinler slaytlar halinde birleştirilecek.
- moviepy veya benzeri bir kütüphane ile video oluşturulacak.

### 7. Kazananı Belirleme
- Basit bir kural sistemi ile kazanan belirlenecek (örneğin, argüman sayısı, kullanıcı oyu, vb.).

### 8. Sonuçların Kaydedilmesi ve Sunulması
- Tüm çıktıların kaydedilmesi ve kullanıcıya sunulması sağlanacak.

---

Her adımda sana detaylı açıklamalar ve kod örnekleriyle rehberlik edeceğim. Şimdi ilk adıma, yani n8n kurulumu ve temel akışın oluşturulmasına geçebiliriz. Hazır olduğunda "devam" yazman yeterli!
