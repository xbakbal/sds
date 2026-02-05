# SF Password Reset - Geliştirici Dokümantasyonu

## 📋 Proje Genel Bakış

**sf-pw-reset**, SAP SuccessFactors kullanıcılarının şifrelerini güvenli bir şekilde sıfırlamasına olanak tanıyan self-servis bir web uygulamasıdır. Uygulama, **SAP Cloud Application Programming (CAP)** model kullanılarak geliştirilmiş olup, SAPUI5 Fiori frontend ve Node.js backend mimarisine sahiptir.

### Temel Özellikler
- ✅ SuccessFactors SCIM API entegrasyonu ile kullanıcı doğrulama
- ✅ SAP OData servisi ile SMS doğrulama kodu gönderimi
- ✅ Çok adımlı güvenli şifre sıfırlama akışı
- ✅ PostgreSQL veritabanı desteği (prod & dev)
- ✅ BTP (Business Technology Platform) üzerinde deployment
- ✅ Responsive Fiori UI/UX

---

## 🏗️ Mimari Yapı

### Teknoloji Stack

| Katman | Teknoloji | Versiyon |
|--------|-----------|----------|
| **Backend Framework** | SAP CAP (Cloud Application Programming) | ^7 |
| **Frontend Framework** | SAPUI5 (Fiori) | Latest |
| **Runtime** | Node.js | Latest LTS |
| **Database** | PostgreSQL | ^1 (via @cap-js/postgres) |
| **Authentication** | XSUAA (SAP Authorization & Trust) | ^3 |
| **API Integration** | SAP Cloud SDK HTTP Client | ^3.23.0 |
| **SMS Service** | SAP OData (Custom Z Service) | - |
| **App Router** | SAP Approuter | Latest |

### Proje Yapısı

```
sf-pw-reset/
├── app/                          # UI Layer (Frontend)
│   ├── sfpwresetui/             # SAPUI5 Fiori Application
│   │   └── webapp/
│   │       ├── controller/      # SAPUI5 Controllers
│   │       │   ├── UserInfo.controller.js
│   │       │   ├── SmsVerification.controller.js
│   │       │   └── NewPassword.controller.js
│   │       ├── view/            # SAPUI5 XML Views
│   │       │   ├── UserInfo.view.xml
│   │       │   ├── SmsVerification.view.xml
│   │       │   └── NewPassword.view.xml
│   │       └── Component.js     # UI5 Component
│   ├── xs-app.json             # App Router Configuration
│   └── router/                  # Routing Configuration
│
├── srv/                         # Service Layer (Backend)
│   ├── service.cds             # CDS Service Definition
│   └── service.js              # Service Implementation (Business Logic)
│
├── db/                          # Data Model Layer
│   └── schema.cds              # Database Schema Definition
│
├── gen/                         # Generated artifacts (build output)
│   ├── srv/                    # Compiled service
│   └── pg/                     # PostgreSQL deployment artifacts
│
├── mta.yaml                    # Multi-Target Application Descriptor
├── package.json                # Node.js Dependencies
├── xs-security.json            # XSUAA Security Configuration
└── README.md                   # Bu Dokümantasyon
```

---

## 🔄 Uygulama Akışı

### Kullanıcı Senaryosu: Şifre Sıfırlama

```mermaid
graph TD
    A[Kullanıcı Uygulamayı Açar] --> B[1️⃣ Kullanıcı Adı & Telefon Girer]
    B --> C{SuccessFactors'da<br/>Kullanıcı Var mı?}
    C -->|Hayır| D[❌ Hata Mesajı]
    C -->|Evet| E{Telefon No<br/>Eşleşiyor mu?}
    E -->|Hayır| D
    E -->|Evet| F[2️⃣ SMS Kodu Gönder]
    F --> G[SMS Doğrulama Ekranı]
    G --> H{SMS Kodu<br/>Doğru mu?}
    H -->|Hayır| I[❌ Hata: Yanlış Kod]
    H -->|Evet| J[3️⃣ Yeni Şifre Ekranı]
    J --> K[Kullanıcı Yeni Şifre Girer]
    K --> L{Şifre SCIM API'ye<br/>Gönderilir}
    L -->|Hata| M[❌ Şifre Geçmişte Kullanılmış]
    L -->|Başarılı| N[✅ Şifre Başarıyla Değiştirildi]
```

### Teknik Akış

#### **1. Kullanıcı Doğrulama (UserInfo.view.xml)**
**Backend Action:** `checkUserName`

```javascript
// srv/service.js - Lines 6-90
// SuccessFactors SCIM API'ye GET request
GET https://ac9jfwmoq.accounts.ondemand.com/scim/Users?filter=userName eq "USERNAME"
Headers:
  - Authorization: Basic [CREDENTIALS]
  - Content-Type: application/scim+json

Response:
{
  "totalResults": 1,
  "Resources": [{
    "id": "user-id-here",
    "userName": "username",
    "phoneNumbers": [{ "type": "work", "value": "+905XXXXXXXXX" }],
    "urn:ietf:params:scim:schemas:extension:enterprise:2.0:User": {
      "organization": "Company Name (COMPANY_CODE)"
    }
  }]
}
```

**Validasyon Kuralları:**
- Kullanıcı adı SuccessFactors'da mevcut olmalı
- Girilen telefon numarası SCIM'deki "work" telefon ile eşleşmeli (format: `+905XXXXXXXXX`)
- Company code organization alanından regex ile extract edilir: `\(([^)]+)\)`

#### **2. SMS Doğrulama (SmsVerification.view.xml)**
**Backend Action:** `sendSms`

```javascript
// srv/service.js - Lines 162-202
// SAP OData SMS Service'e GET request
GET /sap/opu/odata/sap/ZMSF_ODATA_SMS_SRV/SMSSet(
  SIRKET_KODU='COMPANY_CODE',
  TELNO='5XXXXXXXXX',
  MESAJ='SMS DOGRULAMA KODUNUZ: <üretilen_sms_kodu> LUTFEN KIMSEYLE PAYLASMAYINIZ!'
)

// Cloud Connector üzerinden SAP backend'e yönlendirilir
Destination: PROD_SMS_DESTINATION
```

**SMS Kodu Üretimi:**
- 6 haneli rastgele sayı: `Math.floor(100000 + Math.random() * 900000)`
- Frontend'de component model'de saklanır: `/smsCodeFromService`

#### **3. Şifre Güncelleme (NewPassword.view.xml)**
**Backend Action:** `setNewPassword`

```javascript
// srv/service.js - Lines 100-160
// SuccessFactors SCIM API'ye PATCH request
PATCH https://ac9jfwmoq.accounts.ondemand.com/scim/Users/{userId}
Headers:
  - Authorization: Basic [CREDENTIALS]
  - Content-Type: application/scim+json

Body:
{
  "schemas": ["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
  "Operations": [{
    "op": "add",
    "path": "urn:ietf:params:scim:schemas:core:2.0:User:password",
    "value": "YeniŞifre123!"
  }]
}
```

**Şifre Politikası:**
- SuccessFactors şifre politikası uygulanır
- Geçmişte kullanılan şifreler reddedilir (HTTP 400: "Password is already in history")

---

## 💾 Veritabanı Şeması

### Entity Definitions (db/schema.cds)

```cds
namespace sfpwchange;

using { cuid, managed } from '@sap/cds/common';

entity UsageLog : cuid, managed {
    userName    : String;
    displayName : String;
    email       : String;
    phoneNumber : String;
}

entity TempCodes : cuid, managed {
    userName : String;
    tempCode : String;
}
```

**Not:** `cuid` (Common Unique ID) ve `managed` aspectleri otomatik olarak şunları ekler:
- `ID` (UUID primary key)
- `createdAt`, `createdBy`, `modifiedAt`, `modifiedBy` alanları

### Database Profilleri

| Profil | Environment | Credentials |
|--------|-------------|-------------|
| **Default** | Production (Cloud) | AWS RDS PostgreSQL (host: `postgres-46addd8e-7800-4734-9acc-8ac9b646c79d.ce4jcviyvogb.eu-central-1.rds.amazonaws.com`) |
| **sqlite** | Local Development | SQLite (`db.sqlite`) |
| **pglocal** | Local PostgreSQL | localhost:5432, schema: `yasar` |

---

## 🔐 Güvenlik ve Entegrasyonlar

### 1. SuccessFactors SCIM API

**Endpoint:** `https://ac9jfwmoq.accounts.ondemand.com/scim`

**Authentication:** Basic Authentication
- Kodda hardcoded: `Authorization: Basic WEJBS0JBTDptNERwdDNYVzczd3N5dTg=`
- **⚠️ GÜVENLİK NOTU:** Credentials environment variable'a taşınmalı!

**API Operations:**
- `GET /scim/Users?filter=userName eq "..."` - Kullanıcı arama
- `PATCH /scim/Users/{id}` - Şifre güncelleme

### 2. SAP OData SMS Service

**Service:** `ZMSF_ODATA_SMS_SRV` (Custom Z Service)

**Connection:**
- Cloud Connector üzerinden on-premise SAP sistemine bağlantı
- Destination: `PROD_SMS_DESTINATION`
- Connectivity Service: `yasarprod_connectivity`

**Entity Set:** `SMSSet(SIRKET_KODU='...',TELNO='...',MESAJ='...')`

### 3. XSUAA (Authentication)

**Service Plan:** `application`
- Tenant Mode: `dedicated`
- App Name: `sf-pw-reset-xsuaa`
- **⚠️ NOT:** `xs-security.json` boş (public erişim için `authenticationMethod: "none"`)

---

## 🚀 Kurulum ve Çalıştırma

### Ön Gereksinimler

```bash
# Node.js ve npm kurulu olmalı (v18+ önerilir)
node --version
npm --version

# SAP Cloud Foundry CLI (BTP deployment için)
cf --version

# PostgreSQL (local development için, opsiyonel)
psql --version
```

### Lokal Geliştirme Ortamı

#### 1. Bağımlılıkları Yükleme

```bash
# Ana dizinde
npm install

# UI uygulaması için
cd app/sfpwresetui
npm install
cd ../..
```

#### 2. Veritabanı Kurulumu

**SQLite (Hızlı Test):**
```bash
# SQLite profilini kullan (default dev)
cds watch
```

**PostgreSQL (Prod-like):**
```bash
# PostgreSQL servisinin çalıştığından emin ol
# Veritabanını deploy et
npm run pgdeploy

# PostgreSQL profiliyle çalıştır
npm run pg
```

#### 3. Uygulamayı Başlatma

```bash
# Fiori uygulamasını açarak başlat
npm run watch-sfpwresetui

# Veya sadece backend servisi
cds watch
```

**Erişim:**
- UI: http://localhost:4004/app/sfpwresetui/webapp/index.html
- Service Metadata: http://localhost:4004/userpasswordservice/$metadata

### Ortam Değişkenleri (Önerilen)

Güvenlik için aşağıdaki değerleri `.env` dosyasına taşıyın:

```bash
# .env dosyası oluştur (GIT'e ekleme!)
SF_SCIM_BASE_URL=https://ac9jfwmoq.accounts.ondemand.com/scim
SF_SCIM_USERNAME=<username_buraya>
SF_SCIM_PASSWORD=<şifre_buraya>
SMS_DESTINATION_NAME=PROD_SMS_DESTINATION
```

**⚠️ UYARI:** `.gitignore` dosyasına `.env` eklenmiş olmalı!

---

## 📦 Deployment (BTP Cloud Foundry)

### MTA Build ve Deploy

#### 1. MTA Arşivi Oluşturma

```bash
# MTA build
mbt build

# mta_archives/ dizininde .mtar dosyası oluşur
```

#### 2. Cloud Foundry'ye Deploy

```bash
# BTP'ye login
cf login -a https://api.cf.eu10.hana.ondemand.com

# Org ve space seç
cf target -o <ORG_NAME> -s <SPACE_NAME>

# Deploy
cf deploy mta_archives/sf-pw-reset_1.0.0.mtar
```

### MTA Modül Yapısı

| Modül | Tip | Açıklama |
|-------|-----|----------|
| **sf-pw-reset-srv** | nodejs | CAP Backend Service |
| **sf-pw-reset** | approuter.nodejs | Frontend App Router |
| **sf-pw-reset-postgres-deployer** | nodejs | DB Schema Deployer (one-time task) |

### Gerekli BTP Services

```yaml
# mta.yaml'da tanımlanmış resource'lar
resources:
  - sf-pw-reset-auth         # XSUAA Service
  - yasarprod_postgresql     # PostgreSQL DB (existing service)
  - yasarprod_destination    # SAP Destination Service
  - yasarprod_connectivity   # Cloud Connector
```

**Deploy Öncesi Checklist:**
1. ✅ `yasarprod_postgresql` service'i space'de mevcut mu?
2. ✅ `yasarprod_destination` içinde `PROD_SMS_DESTINATION` tanımlı mı?
3. ✅ Cloud Connector yapılandırması aktif mi?
4. ✅ XSUAA credential'ları güncel mi?

---

## 🛠️ Geliştirme Notları

### CDS Service Actions

| Action | Parameters | Return | Açıklama |
|--------|-----------|--------|----------|
| `checkUserName` | `username: String` | `{success, id, userName, phoneNumber, companyCode}` | SF kullanıcı doğrulama |
| `setNewPassword` | `id: String, password: String` | `{success, message}` | Şifre güncelleme |
| `sendSms` | `companycode: String, phonenumber: String, message: String` | `{success, message}` | SMS gönderimi |

### UI Component Model Yapısı

```javascript
// Component.js - Shared model
{
  id: "",              // SF User ID (SCIM'den gelir)
  username: "",        // Kullanıcı adı
  phone: "",           // Telefon numarası (10 haneli)
  smsCode: "",         // Kullanıcının girdiği SMS kodu
  smsCodeFromService: "", // Backend'den gelen asıl SMS kodu
  password: "",        // Yeni şifre
  confirmPassword: "", // Şifre tekrar
  companyCode: ""      // Şirket kodu (SMS için gerekli)
}
```

---

## 🐛 Bilinen Sorunlar ve Çözümler

### 1. SMS Gönderimi Başarısız Oluyor

**Belirtiler:**
```javascript
Error calling service: [Error details]
```

**Çözüm:**
- Cloud Connector'ın çalıştığını kontrol et
- `PROD_SMS_DESTINATION` destination'ının doğru yapılandırıldığından emin ol
- SAP backend'deki `ZMSF_ODATA_SMS_SRV` servisinin aktif olduğunu kontrol et

### 2. Şifre Geçmişte Kullanılmış Hatası

**Belirtiler:**
```
"Password is already in history."
```

**Çözüm:**
- Kullanıcıdan farklı bir şifre denemesini iste
- SuccessFactors şifre politikasını kontrol et (Admin Center > Password Policy)

### 3. PostgreSQL Bağlantı Hatası (Local)

**Belirtiler:**
```
Error: connect ECONNREFUSED ::1:5432
```

**Çözüm:**
```bash
# PostgreSQL servisini başlat
brew services start postgresql  # macOS
sudo service postgresql start   # Linux

# Veritabanını yeniden deploy et
npm run pgdeploy
```

---

## 🧪 Test Senaryoları

### Manuel Test Checklist

#### Test 1: Kullanıcı Doğrulama
1. Uygulamayı aç: http://localhost:4004/app/sfpwresetui/webapp/index.html
2. Geçerli bir SF kullanıcı adı gir (örn: `testuser`)
3. Kullanıcının SF'deki telefon numarasını gir (format: `5XXXXXXXXX`)
4. **Beklenen:** SMS doğrulama ekranına geçiş

#### Test 2: Hatalı Telefon Numarası
1. Doğru kullanıcı adı gir
2. **Hatalı** telefon numarası gir
3. **Beklenen:** "Doğrulama başarısız! Lütfen bilgilerinizi kontrol ediniz!"

#### Test 3: SMS Doğrulama
1. SMS ekranına kadar ilerle
2. Gelen SMS kodunu gir (6 haneli)
3. **Beklenen:** Yeni şifre ekranına geçiş

#### Test 4: Şifre Sıfırlama
1. Yeni şifre ekle (SF şifre politikasına uygun)
2. Aynı şifreyi tekrar gir
3. **Beklenen:** "Şifre başarıyla değiştirildi" mesajı

---

## 📊 Performans ve Optimizasyon

### API Call Süreleri (Ortalama)

| Endpoint | Süre | Açıklama |
|----------|------|----------|
| `checkUserName` | ~500ms | SCIM API network latency |
| `sendSms` | ~1-2s | Cloud Connector + SAP backend |
| `setNewPassword` | ~400ms | SCIM PATCH operation |

---

## 📚 Ek Kaynaklar

### SAP CAP Dokümantasyonu
- [CAP Official Docs](https://cap.cloud.sap/docs/)
- [CDS Language Reference](https://cap.cloud.sap/docs/cds/)
- [PostgreSQL Adapter](https://cap.cloud.sap/docs/guides/databases-postgres)

### SAPUI5 / Fiori
- [SAPUI5 SDK](https://sapui5.hana.ondemand.com/)
- [Fiori Design Guidelines](https://experience.sap.com/fiori-design/)

### BTP Deployment
- [MTA Developer Guide](https://help.sap.com/docs/BTP/65de2977205c403bbc107264b8eccf4b/4050fee4c469498ebc31b10f2ae15ff2.html)
- [Cloud Foundry CLI](https://docs.cloudfoundry.org/cf-cli/)

### SuccessFactors SCIM API
- [SCIM 2.0 Protocol](https://datatracker.ietf.org/doc/html/rfc7644)
- [SF API Documentation](https://help.sap.com/docs/SAP_SUCCESSFACTORS_PLATFORM/d599f15995d348a1b45ba5603e2aba9b/03b7b9306e694c45921e70644c6c5295.html)

---

## 👥 Destek ve İletişim

### Yaygın Sorular

**S: SCIM credential'ları nasıl güncellenir?**
A: `srv/service.js` dosyasında hardcoded base64 credential yerine environment variable kullanın:
```javascript
const auth = `Basic ${Buffer.from(`${process.env.SF_USERNAME}:${process.env.SF_PASSWORD}`).toString('base64')}`
```

**S: Farklı bir SMS servisi nasıl entegre edilir?**
A: `srv/service.js` içinde `sendSms` action'ını güncelle, yeni service'in API contract'ına göre düzenle.

**S: Deployment sırasında PostgreSQL hatası alıyorum?**
A: `mta.yaml` içinde `sf-pw-reset-postgres` resource'ının `service-name` değerini BTP'deki mevcut service ile eşleştir.

---

## 📝 Geliştirme Geçmişi

### v1.0.0 (Mevcut Versiyon)
- ✅ SuccessFactors SCIM entegrasyonu
- ✅ SAP OData SMS servisi entegrasyonu
- ✅ 3 adımlı şifre sıfırlama UI
- ✅ PostgreSQL database support
- ✅ BTP Cloud Foundry deployment
---

## ⚖️ Lisans

Bu proje Yaşar Holding için, Solvia Digital Solutions tarafından geliştirilmiştir. Tüm hakları saklıdır.

**Private Repository** - Unauthorized distribution is prohibited.

---

**Son Güncelleme:** 2025-01-01
**Versiyon:** 1.0.0
**Geliştirici:** Solvia Development Team
