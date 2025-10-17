
---

## Yapılan Değişiklikler

### ✅ 1. Config Ayarları (`lib/config.js`)

**Eklenenler:**
```javascript
// SMS Configuration
export const SMS_CONFIG = {
  enabled: true,
  codeLength: 6,
  codeExpireMinutes: 10,
  provider: 'netgsm' // 'netgsm' veya 'development'
};

// Netgsm API Configuration
export const NETGSM_CONFIG = {
  usercode: '', // Netgsm abone numaranız (850xxxxxxx)
  password: '', // API yetkisi tanımlı alt kullanıcı şifresi
  msgheader: '', // Sistemde tanımlı mesaj başlığınız (3-11 karakter)
  apiUrl: 'https://api.netgsm.com.tr/sms/send/otp',
  template: 'Doğrulama kodunuz: {CODE}\n\nGüvenliğiniz için bu kodu kimseyle paylaşmayın.\n\nTEST DENEME ASISTAN'
};
```

**Özellikler:**
- `SMS_CONFIG.provider`: `'netgsm'` → Gerçek SMS | `'development'` → Console log
- `SMS_CONFIG.codeLength`: OTP kod uzunluğu (varsayılan: 6)
- `SMS_CONFIG.codeExpireMinutes`: Kodun geçerlilik süresi (varsayılan: 10 dakika)
- `NETGSM_CONFIG.template`: SMS mesaj şablonu (`{CODE}` placeholder ile)

---

### ✅ 2. Background Script (`background/supabase-handler.js`)

#### 2.1. Config Import
```javascript
// SMS config
const SMS_CONFIG = {
  enabled: true,
  codeLength: 6,
  codeExpireMinutes: 10,
  provider: 'netgsm' // 'netgsm' veya 'development'
};

// Netgsm API Configuration
const NETGSM_CONFIG = {
  usercode: '', // Netgsm credentials
  password: '',
  msgheader: '',
  apiUrl: 'https://api.netgsm.com.tr/sms/send/otp',
  template: 'Doğrulama kodunuz: {CODE}\n\nGüvenliğiniz için bu kodu kimseyle paylaşmayın.\n\nTEST DENEME ASISTAN'
};

// OTP kodlarını geçici olarak sakla (memory)
const otpStore = new Map(); // { phone: { code, expiresAt } }
```

#### 2.2. Yeni Fonksiyonlar

**`generateOTPCode(length = 6)`**
- Random 6 haneli OTP kodu üretir
- Örnek: `"123456"`, `"789012"`

**`cleanupExpiredOTP()`**
- Süresi dolmuş OTP kodlarını memory'den temizler
- Her SMS gönderimi ve doğrulama öncesi çağrılır

**`sendNetgsmOTP(phone, code)`**
- Netgsm API'ye XML POST isteği gönderir
- Hata kodlarını kontrol eder (20, 30, 40-41, 50-52, 60, 70, 100)
- Başarılı ise `bulkID` döner

**XML Request Formatı:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<mainbody>
  <header>
    <company dil="TR">Netgsm</company>
    <usercode>850xxxxxxx</usercode>
    <password>xxxxxx</password>
    <type>1:n</type>
    <msgheader>MSGHEADER</msgheader>
  </header>
  <body>
    <msg><![CDATA[Doğrulama kodunuz: 123456...]]></msg>
    <no>905xxxxxxxxx</no>
  </body>
</mainbody>
```

#### 2.3. Güncellenen Action'lar

**`sendVerificationSMS`**
```javascript
case 'sendVerificationSMS': {
  const { phone } = data;

  // 1. Expired OTP'leri temizle
  cleanupExpiredOTP();

  // 2. Yeni OTP kodu üret (6 haneli)
  const code = generateOTPCode(SMS_CONFIG.codeLength);

  // 3. Expire time hesapla (10 dakika sonra)
  const expiresAt = Date.now() + (SMS_CONFIG.codeExpireMinutes * 60 * 1000);

  // 4. OTP'yi memory'de sakla
  otpStore.set(phone, { code, expiresAt });

  // 5. Provider'a göre SMS gönder
  if (SMS_CONFIG.provider === 'netgsm') {
    try {
      const result = await sendNetgsmOTP(phone, code);
      return {
        success: true,
        message: 'Verification code sent via Netgsm',
        provider: 'netgsm',
        bulkId: result.bulkId
      };
    } catch (error) {
      // Netgsm başarısız olursa development mode'a düş
      return {
        success: true,
        message: 'Verification code sent (development mode)',
        provider: 'development',
        debug_code: code,
        error: error.message
      };
    }
  } else {
    // Development mode
    return {
      success: true,
      message: 'Verification code sent (development mode)',
      provider: 'development',
      debug_code: code
    };
  }
}
```

**`verifyPhoneCode`**
```javascript
case 'verifyPhoneCode': {
  const { phone, code } = data;

  // 1. Expired OTP'leri temizle
  cleanupExpiredOTP();

  // 2. OTP'yi memory'den al
  const storedOTP = otpStore.get(phone);

  if (!storedOTP) {
    throw new Error('Verification code expired or not found');
  }

  // 3. Kod kontrolü
  if (storedOTP.code !== code) {
    throw new Error('Invalid verification code');
  }

  // 4. Expire kontrolü
  if (Date.now() > storedOTP.expiresAt) {
    otpStore.delete(phone);
    throw new Error('Verification code expired');
  }

  // 5. Kod doğru, memory'den sil
  otpStore.delete(phone);

  // 6. User profile'ı güncelle (phone_verified: true)
  const { data: { user } } = await client.auth.getUser();
  if (user) {
    await client
      .from('user_profiles')
      .update({ phone_verified: true })
      .eq('id', user.id);
  }

  return { success: true, verified: true };
}
```

---

## Netgsm Hata Kodları

| Kod | Açıklama |
|-----|----------|
| `20` | Mesaj metni yok |
| `30` | Geçersiz kullanıcı adı veya şifre |
| `40` | Mesaj başlığı sistemde kayıtlı değil |
| `41` | Mesaj başlığı kullanıcıya ait değil |
| `50` | Geçersiz birim (mesaj başına düşecek kredi) |
| `51` | Hatalı sorgu formatı |
| `52` | Gönderilen SMS kullanıcının limitini aştı |
| `60` | Hatalı telefon numarası |
| `70` | Hatalı sorgulama |
| `100` | Sistem hatası |

**Başarılı yanıt:** Sayısal `bulkID` (örn: `123456789`)

---

## Kurulum Adımları

### 1. Netgsm Hesabı Oluştur
1. [Netgsm](https://www.netgsm.com.tr) hesabı oluştur
2. SMS kredisi yükle
3. **API yetkisi tanımlı alt kullanıcı** oluştur

### 2. Mesaj Başlığı (msgheader) Tanımla
1. Netgsm paneline giriş yap
2. **SMS Ayarları → Mesaj Başlıkları** bölümüne git
3. Yeni başlık ekle (3-11 karakter, örn: `TRENDYOL`)
4. Onay bekle (genellikle 1-2 iş günü)

### 3. Config Dosyasını Güncelle

**`lib/config.js`**
```javascript
export const NETGSM_CONFIG = {
  usercode: '8505551234', // Netgsm abone numaranız
  password: 'your_password', // API yetkisi tanımlı alt kullanıcı şifresi
  msgheader: 'TRENDYOL', // Sistemde tanımlı mesaj başlığınız
  apiUrl: 'https://api.netgsm.com.tr/sms/send/otp',
  template: 'Doğrulama kodunuz: {CODE}\n\nGüvenliğiniz için bu kodu kimseyle paylaşmayın.\n\n TEST DENEME ASISTAN'
};
```

**`background/supabase-handler.js`**
```javascript
const NETGSM_CONFIG = {
  usercode: '8505551234', // Aynı bilgileri buraya da gir
  password: 'your_password',
  msgheader: 'TRENDYOL',
  apiUrl: 'https://api.netgsm.com.tr/sms/send/otp',
  template: 'Doğrulama kodunuz: {CODE}\n\nGüvenliğiniz için bu kodu kimseyle paylaşmayın.\n\nTEST DENEME ASISTAN'
};
```

### 4. Provider Ayarını Kontrol Et

**Netgsm kullanmak için:**
```javascript
const SMS_CONFIG = {
  provider: 'netgsm' // ✅ Gerçek SMS gönderimi
};
```

**Development mode için:**
```javascript
const SMS_CONFIG = {
  provider: 'development' // ✅ Console'a kod yazdırır, SMS göndermez
};
```

---

## Test Senaryoları

### 1. Kayıt Testi (Netgsm Mode)
```
1. Chrome Extension'ı yükle
2. Herhangi bir özelliğe tıkla (giriş yapılmamış)
3. "Hemen Kayıt Olun" → Email, Şifre, Telefon gir
4. "Kayıt Ol" butonuna tıkla
   ✅ Netgsm üzerinden SMS gönderilir
   ✅ Console: "SMS sent successfully, bulk ID: 123456789"
5. Telefona gelen 6 haneli kodu gir
   ✅ Kod doğru → Telefon doğrulanır
   ✅ Email doğrulama modalı açılır
```

### 2. Hatalı Kod Testi
```
1. Kayıt ol → SMS gönder
2. Yanlış kod gir (örn: "000000")
   ❌ "Invalid verification code" hatası
3. 10 dakika bekle
4. Doğru kodu gir
   ❌ "Verification code expired" hatası
```

### 3. Development Mode Testi
```javascript
// SMS_CONFIG.provider = 'development' yap

1. Kayıt ol → SMS gönder
2. Console'da kodu gör:
   📱 Development mode - Code: 123456
3. Aynı kodu gir
   ✅ Telefon doğrulanır
```

### 4. Netgsm Hata Testi
```
1. NETGSM_CONFIG.usercode = '' (boş bırak)
2. Kayıt ol → SMS gönder
   ❌ Console: "Netgsm credentials not configured"
   ✅ Fallback → Development mode'a düşer
   ✅ Console'da kod görünür
```

---

## Console Log Örnekleri

### Başarılı SMS Gönderimi
```
📱 Generated OTP for +905551234567: 123456 (expires in 10m)
📤 Sending Netgsm OTP to: 905551234567
📥 Netgsm response: 987654321
✅ SMS sent successfully, bulk ID: 987654321
✅ Netgsm SMS sent: { success: true, bulkId: '987654321' }
```

### Hatalı Kredensiyel
```
📱 Generated OTP for +905551234567: 123456 (expires in 10m)
📤 Sending Netgsm OTP to: 905551234567
📥 Netgsm response: 30
❌ Netgsm SMS send error: Netgsm Error 30: Geçersiz kullanıcı adı veya şifre
❌ Netgsm SMS failed, falling back to development mode
```

### Kod Doğrulama
```
✅ OTP verified and removed from store: +905551234567
✅ Profile updated - phone_verified: true
✅ Phone verified for +905551234567
```

### Hatalı Kod
```
❌ Invalid OTP code: { expected: '123456', received: '000000' }
```

### Süresi Dolmuş Kod
```
❌ OTP expired for phone: +905551234567
🗑️ Expired OTP removed for +905551234567
```

---

## Güvenlik Notları

### 1. Credentials Güvenliği
**UYARI:** `NETGSM_CONFIG` bilgileri hassas verilerdir!

- ✅ Production'da environment variable kullan
- ✅ Git'e commit etme (`.gitignore` ekle)
- ✅ Extension yayınlanmadan önce credentials'ları temizle

**Örnek `.gitignore`:**
```
lib/config.js
background/supabase-handler.js
```

### 2. Rate Limiting
- Netgsm API'de rate limit yoktur, ancak kredi limiti vardır
- SMS başına kredi düşer
- `SMS_CONFIG.codeExpireMinutes` ile gereksiz SMS gönderimini engelle

### 3. OTP Storage
- OTP kodları **memory'de** saklanır (`otpStore` Map)
- Background script restart olursa kodlar silinir
- Alternatif: Database'de sakla (daha güvenli, persistent)

### 4. Phone Format
- Telefon numaraları `+90` ile başlamalı
- Netgsm'e gönderilirken `+` kaldırılır → `905551234567`
- Sadece rakamlar gönderilir (boşluk, tire vb. temizlenir)

---

## Gelecek Geliştirmeler

### 1. Database Storage
```sql
CREATE TABLE phone_verifications (
  phone TEXT PRIMARY KEY,
  code TEXT NOT NULL,
  expires_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);
```

### 2. Retry Mekanizması
```javascript
// 3 deneme hakkı
let attempts = 3;
if (invalidCode && attempts > 0) {
  attempts--;
  throw new Error(`Invalid code. ${attempts} attempts left.`);
}
```

### 3. SMS Template Yönetimi
```javascript
// Farklı diller için template
const SMS_TEMPLATES = {
  tr: 'Doğrulama kodunuz: {CODE}\n\nTEST DENEME ASISTAN',
  en: 'Your verification code: {CODE}\n\nTEST DENEME ASISTAN'
};
```

### 4. Metrics & Monitoring
```javascript
// SMS gönderim istatistikleri
const smsMetrics = {
  sent: 0,
  failed: 0,
  verified: 0,
  expired: 0
};
```

---

## Sorun Giderme

### Sorun: SMS Gelmiyor

**Çözüm:**
1. Console'da `bulkID` kontrolü yap → Varsa SMS gönderildi
2. Netgsm panelde SMS loglarını kontrol et
3. Mesaj başlığı (msgheader) onaylandı mı?
4. SMS kredisi yeterli mi?

### Sorun: "Geçersiz kullanıcı adı veya şifre" (Kod 30)

**Çözüm:**
1. `NETGSM_CONFIG.usercode` doğru mu? (850xxxxxxx formatında)
2. `NETGSM_CONFIG.password` API yetkisi tanımlı mı?
3. Alt kullanıcı şifresi doğru mu?

### Sorun: "Mesaj başlığı sistemde kayıtlı değil" (Kod 40)

**Çözüm:**
1. Netgsm panelde mesaj başlığı tanımlı mı?
2. Onay aldınız mı? (1-2 iş günü sürer)
3. `NETGSM_CONFIG.msgheader` doğru mu? (3-11 karakter)

### Sorun: "Verification code expired"

**Çözüm:**
1. `SMS_CONFIG.codeExpireMinutes` artır (örn: 15 dakika)
2. Yeni kod iste
3. Database storage kullan (restart olsa bile kod kaybolmaz)

---

## Değiştirilen Dosyalar

### 1. `lib/config.js`
- ✅ `SMS_CONFIG` eklendi
- ✅ `NETGSM_CONFIG` eklendi

### 2. `background/supabase-handler.js`
- ✅ `SMS_CONFIG` import
- ✅ `NETGSM_CONFIG` import
- ✅ `otpStore` Map eklendi
- ✅ `generateOTPCode()` fonksiyonu
- ✅ `cleanupExpiredOTP()` fonksiyonu
- ✅ `sendNetgsmOTP()` fonksiyonu
- ✅ `sendVerificationSMS` action güncellendi
- ✅ `verifyPhoneCode` action güncellendi

---

## Sonuç

✅ **Netgsm OTP API entegrasyonu tamamlandı!**

Artık telefon doğrulama sistemi:
- ✅ Gerçek SMS gönderebiliyor (Netgsm)
- ✅ Development mode destekliyor (console log)
- ✅ OTP kodlarını memory'de saklıyor
- ✅ Expire time kontrolü yapıyor (10 dakika)
- ✅ Hata kodlarını handle ediyor
- ✅ Fallback mekanizması var (Netgsm → Development)

**Not:** Extension'ı yayınlamadan önce:
1. `NETGSM_CONFIG` credentials'larını doldur
2. `SMS_CONFIG.provider = 'netgsm'` yap
3. Test et
4. Credentials'ları commit etme!
