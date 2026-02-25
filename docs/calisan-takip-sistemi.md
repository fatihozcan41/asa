# Çalışan Giriş-Çıkış, Müşteri Ziyareti ve Yönetici Raporlama Sistemi

## 1) Amaç
Bu doküman; ofiste çalışan, sahaya çıkan ve gün içinde hem ofis hem müşteri ziyareti yapan personelin
**QR kod + mobil uygulama** ile zaman/konum bazlı takibini yapmak için iş gereksinimlerini tanımlar.

Hedefler:
- Çalışanların işe başlama, işten çıkma, müşteriye gidiş ve müşteriden dönüş hareketlerini kayıt altına almak.
- Kayıtları tarih, saat, konum ve olay tipi bazında merkezi olarak toplamak.
- Yöneticiler için günlük/haftalık/aylık analiz raporları üretmek.
- Sisteme kullanıcı adı + şifre ile güvenli giriş sağlamak.

## 2) Kullanıcı Rolleri

### 2.1 Çalışan
- Mobil uygulama/web ile kullanıcı adı + şifre kullanarak giriş yapar.
- Kendi adına tanımlı QR kodu okutur.
- Olay tipini seçer:
  - İşe Başlangıç
  - İşten Çıkış
  - Müşteriye Çıkış
  - Müşteriden Dönüş
- Sistem; olay zamanı, koordinat bilgisi, cihaz bilgisi ve doğrulama durumunu kaydeder.

### 2.2 Yönetici
- Tüm çalışan kayıtlarını görüntüler.
- Departman/çalışan/tarih filtreli raporlar alır.
- Geç kalma, eksik kayıt, fazla mesai, saha ziyaret performansı gibi metrikleri inceler.

### 2.3 Sistem Yöneticisi (Opsiyonel)
- Kullanıcı, rol, müşteri lokasyonu, vardiya ve tolerans ayarlarını yönetir.
- QR kod üretim ve iptal süreçlerini yönetir.

## 3) Temel Fonksiyonlar

### 3.1 Kimlik Doğrulama
- Zorunlu kullanıcı adı + şifre.
- Güçlü şifre politikası (minimum uzunluk, karmaşıklık, süreli yenileme).
- Oturum zaman aşımı ve isteğe bağlı 2FA.

### 3.2 QR Kod Akışı
- Her çalışan için benzersiz QR kod (ya da şirket giriş noktası + müşteri lokasyonu bazlı QR).
- QR okutulduğunda aşağıdaki veri toplanır:
  - `employee_id`
  - `event_type`
  - `event_time` (sunucu saati)
  - `device_time` (istemci saati)
  - `latitude`, `longitude`, `accuracy`
  - `location_source` (GPS/Wi-Fi/Cell)
  - `qr_id`
  - `device_id` / `app_version`
- QR tokenı tek kullanımlık veya kısa süreli imzalı olabilir (güvenlik için önerilir).

### 3.3 Konum Doğrulama
- Ofis olayları için ofis geofence kontrolü.
- Müşteri olayları için müşteri lokasyonu geofence kontrolü.
- Geofence dışı olaylarda:
  - "Şüpheli" etiketi
  - Yöneticiye uyarı
  - Çalışandan açıklama talebi

### 3.4 Olay Eşleştirme Kuralları
- İşe Başlangıç -> İşten Çıkış eşleşmeli.
- Müşteriye Çıkış -> Müşteriden Dönüş eşleşmeli.
- Eksik eşleşmeler "eksik kayıt" olarak raporlanmalı.
- Aynı tip olayın kısa sürede tekrarında çift kayıt önleme (debounce/idempotency).

### 3.5 Yönetici Raporları
- Günlük devam durumu (kim geldi, kim gelmedi, geç kalanlar).
- Çalışan bazlı çalışma süresi.
- Müşteri ziyaret sayısı, süresi, lokasyon uygunluğu.
- Departman/ekip bazlı performans kıyasları.
- Aykırı hareketler (geofence dışı, çakışan kayıt, eksik çift).

## 4) Örnek Ekranlar
- Giriş ekranı (kullanıcı adı + şifre).
- QR okutma ekranı (kamera + olay tipi seçimi).
- Çalışan geçmişi ekranı (bugün/hafta olay listesi).
- Yönetici paneli:
  - Özet KPI kartları
  - Detay filtreli tablo
  - Grafikler (trend, dağılım)
  - CSV/Excel/PDF dışa aktarım

## 5) Veri Modeli (Öneri)

### 5.1 `employees`
- id
- full_name
- username
- password_hash
- role (employee/manager/admin)
- department
- is_active

### 5.2 `locations`
- id
- name
- type (office/customer)
- latitude
- longitude
- geofence_radius_m

### 5.3 `qr_codes`
- id
- employee_id (opsiyonel, tasarıma bağlı)
- location_id (opsiyonel)
- token_hash
- expires_at
- is_active

### 5.4 `attendance_events`
- id
- employee_id
- event_type
- event_time_utc
- device_time
- latitude
- longitude
- accuracy_m
- qr_id
- location_validation_status (valid/suspicious/invalid)
- created_at

### 5.5 `event_explanations`
- id
- event_id
- employee_id
- explanation_text
- created_at

## 6) İş Kuralları (Özet)
- Sunucu saatini esas al.
- Offline kayıt desteklenebilir; online olunca senkronize edilir.
- Aynı çalışan için aynı dakika içinde aynı olay tipi tek kayıt kabul edilebilir.
- Çalışan yalnızca kendi olaylarını görebilir.
- Yönetici, yetkili olduğu ekip/departmanı görebilir.

## 7) Güvenlik ve KVKK/Uyum
- Şifreler geri döndürülemez hash ile saklanmalı (örn. Argon2/Bcrypt).
- Hassas veriler (konum dahil) şifreli taşıma (TLS) ve yetkili erişim ile korunmalı.
- Erişim logları tutulmalı.
- Veri saklama süresi politikası belirlenmeli (ör. 2 yıl).
- Açık rıza / aydınlatma metni süreçleri kurum hukuk birimiyle netleştirilmeli.

## 8) KPI ve Analiz Önerileri
- Ortalama işe başlama saati
- Geç kalma oranı
- Eksik kayıt oranı
- Toplam saha ziyaret sayısı
- Ziyaret başına ortalama süre
- Geofence uyum oranı
- Çalışan bazlı aylık toplam çalışma süresi

## 9) Uygulama Yol Haritası

### Faz 1 (MVP)
- Giriş (username/password)
- QR okutma + olay kaydı
- Basit yönetici paneli (tablo + filtre)

### Faz 2
- Geofence doğrulama
- Şüpheli kayıt workflow'u
- CSV/PDF rapor dışa aktarım

### Faz 3
- Gelişmiş dashboard ve KPI
- Bildirimler (geç kalma/eksik kayıt)
- ERP/İK entegrasyonları

## 10) Kabul Kriterleri
- Çalışan başarıyla giriş yapabilmeli.
- QR okutulduğunda olay kaydı 2 saniye içinde oluşmalı.
- Olay kaydı zaman + konum + olay tipi ile raporda görünmeli.
- Yönetici filtrelerle rapor alabilmeli.
- Geofence dışı kayıtlar "şüpheli" işaretlenebilmeli.
