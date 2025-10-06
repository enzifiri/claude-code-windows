# Claude Code Kurulum Rehberi DOA SKOOL COMMUNİTY ÖZEL (Windows)

## Neden Claude Code kullanmalıyız?

💰 Maliyet Avantajı

- **Bolt/v0:** Token başına ücretlendirme, hızlı tükenen krediler
- **Claude Code:** Claude Pro aboneliği ile kullanım (aylık ~$20 KAMPANYA İLE 10$)
- **Kampanya Linki** Dikkat!, Daha önce claudeye giriş yapmadığınız cihaz ve mail ile üyelik oluşturun, Gerekirse farklı telefondan mobil veri ile deneyin [10$ Claude Üyeliği için Tıkla](https://claude.ai/alex)
- **Stratejim:** İlk taslağı Bolt/v0'da oluştur, özelleştirmeleri Claude Code'da yap, deployment için tekrar Bolt'a yükle

## 💡 Önerilen İş Akışı (Lovable gibi sistemler için de geçerlidir)

1. Bolt & v0 kullanarak önce projenizin dosyalarını oluşturun
2. Projenizi indirin ve ZIP'ten çıkartın
3. Claude Code'u proje klasörünüzde çalıştırın ve eklemek istediğiniz özellikleri ekleyin
4. Özelleştirmeleriniz bitince Bolt'a geri dönün ve proje dosyalarını yükleyip kaydedin (0 token)

## 🚀 Kurulum Adımları

### 1. Node.js Kurulumu

**Windows Installer (.msi) ile kurulum:**

1. Aşağıdaki bağlantıdan Node.js kurulum dosyasını indirin:
   ```
   https://nodejs.org/dist/v22.20.0/node-v22.20.0-x64.msi
   ```

2. İndirilen `.msi` dosyasını çalıştırın

3. Kurulum sihirbazında "Next" butonlarına tıklayarak ilerleyin

4. Lisans anlaşmasını kabul edin

5. Kurulum tamamlandığında bilgisayarınızı yeniden başlatın

6. Kurulumu doğrulamak için **Komut İstemi** (CMD) veya **PowerShell** açın ve şunu yazın:
   ```bash
   node --version
   ```
   
   Ekranda `v22.20.0` gibi bir sürüm numarası görecekseniz kurulum başarılı demektir.

### 2. Claude Code Kurulumu

1. **Komut İstemi** (CMD) veya **PowerShell**'i **yönetici olarak** açın

2. Aşağıdaki komutu çalıştırın:
   ```bash
   npm install -g @anthropic-ai/claude-code
   ```

3. Kurulum tamamlandığında şu komutla doğrulama yapın:
   ```bash
   claude-code --version
   ```

### 3. Git Kontrolü

Claude Code, Git'in sisteminizde kurulu olmasını gerektirir. Git kurulu değilse:

1. [Git for Windows](https://git-scm.com/download/win) adresinden Git'i indirin ve kurun

2. Kurulumu doğrulamak için:
   ```bash
   git --version
   ```

### 4. İlk Çalıştırma ve Tarayıcı ile Giriş

1. Proje klasörünüze gidin:
   ```bash
   cd C:\Users\YourName\Projects\MyProject
   ```

2. Claude Code'u başlatın:
   ```bash
   claude-code
   ```

3. İlk çalıştırmada Claude Code sizi **tarayıcıya yönlendirecek**

4. Tarayıcıda Claude hesabınıza giriş yapın

5. Yetkilendirmeyi onaylayın

6. Başarılı giriş sonrası terminal'e geri dönün ve Claude Code kullanıma hazır!

> ℹ️ **Not:** API anahtarı gerekmez! Claude Code, tarayıcı üzerinden kimlik doğrulaması yapar ve oturumunuzu güvenli şekilde saklar.

## 📖 Kullanım

1. Proje klasörünüze gidin:
   ```bash
   cd C:\Users\YourName\Projects\MyProject
   ```

2. Claude Code'u başlatın:
   ```bash
   claude-code
   ```

3. Claude'a ne yapmak istediğinizi anlatın ve kodlamaya başlayın!


## 🔧 Sorun Giderme

**Node.js tanınmıyor hatası:**
- Bilgisayarınızı yeniden başlatın
- PATH ortam değişkenini kontrol edin

**npm komutu çalışmıyor:**
- Node.js'in düzgün kurulduğundan emin olun
- Komut İstemi'ni yönetici olarak çalıştırmayı deneyin

**Git bulunamadı hatası:**
- Git'in kurulu olduğundan emin olun
- Komut İstemi'ni kapatıp tekrar açın

**Tarayıcı açılmıyor:**
- Manuel olarak gösterilen URL'yi tarayıcınıza kopyalayın
- Claude.ai'da oturum açtığınızdan emin olun

**Kimlik doğrulama hatası:**
- Claude Pro aboneliğinizin aktif olduğunu kontrol edin
- Tarayıcıda Claude oturumunuzu kapatıp tekrar açın
- `claude-code` komutunu tekrar çalıştırın

## 📚 Daha Fazla Bilgi

- [Claude Code Dokümantasyonu](https://docs.claude.com/en/docs/claude-code)
- [Anthropic Destek](https://support.claude.com)

## 🤝 Katkıda Bulunma

Sorularınız veya önerileriniz için issue açabilirsiniz!

## 📝 Lisans

Bu rehber MIT lisansı altında paylaşılmaktadır.

---

**İyi Kodlamalar! 🚀**


**İyi Kodlamalar! 🚀**
