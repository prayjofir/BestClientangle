# BestClientangle — Agents Guide

Bu dosya, projeye yeni özellik eklerken veya yeni sürüm çıkarken referans olarak kullanılır.

---

## 📁 Proje Yapısı

```
BestClientangle/
├── src/
│   ├── engine/
│   │   ├── shared/
│   │   │   ├── config_variables_bestclient.h   ← BestClient config değişkenleri (bc_*)
│   │   │   └── config_variables_tclient.h      ← TClient config değişkenleri (tc_*)
│   │   └── client/
│   │       └── discord.cpp                     ← Discord RPC implementasyonu
│   └── game/
│       └── client/
│           └── components/
│               ├── controls.h / controls.cpp    ← Aim angle bind sistemi burada
│               ├── bestclient/
│               │   └── menus_bestclient.cpp     ← BestClient ayar menüleri
│               └── tclient/
│                   └── menus_tclient.cpp        ← TClient ayar menüleri (Discord toggle)
├── ddnet-libs/
│   └── discord/
│       ├── include/discord_game_sdk.h
│       └── linux/lib64/discord_game_sdk.so
└── build/                                       ← cmake çıktısı, buradan çalıştır
```

---

## 🔧 Build Sistemi

### İlk Kurulum
```bash
git clone https://github.com/prayjofir/BestClientangle.git
cd BestClientangle
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -DDISCORD=ON -DDISCORD_DYNAMIC=ON \
  -DDISCORDSDK_LIBRARY=../ddnet-libs/discord/linux/lib64/discord_game_sdk.so \
  -DDISCORDSDK_INCLUDEDIR=../ddnet-libs/discord/include
cd ..
```

### Build & Çalıştır
```bash
cmake --build build --target DDNet -j$(nproc)
./build/DDNet
```

### Commit & Push
```bash
git add <dosyalar>
git commit -m "feat/fix: açıklama"
git push
```

---

## ✨ Eklenen Özellikler

### 1. Aim Angle Bind (`+aim_angle` / `+aim_angle2`)

**Amaç:** Belirli bir açıya smooth hareketle aim ettikten sonra sol/sağ+hook çalıştırır.

**Config Değişkenleri** (`config_variables_bestclient.h`):
```cpp
bc_aim_angle      // Sol taraf açısı (örn: -214)
bc_aim_angle2     // Sağ taraf açısı (örn: -590)
bc_aim_angle_speed // Aim hareket hızı (1-1000, varsayılan: 25)
```

**Açı Hesabı:** `AngleRad = bc_aim_angle / 256.0f` (radyan)
- Açı = debug HUD'da görünen "Angle" değeri
- Sağ taraf = sol tarafın Y-ekseni yansıması: `atan2(-sin(L), -cos(L)) * 256`

**Nasıl Çalışır:**
1. Tuşa basılınca `ConKeyAimAngle` çağrılır
2. Eğer aim hedefe 15 birimden uzaksa: `settingPress=true`, hareket başlar
3. `OnRender`'da her frame aim hedefe doğru `bc_aim_angle_speed` kadar hareket eder
4. `SnapInput`'ta `settingPress=true` olduğu sürece direction+hook sıfırlanır (blok)
5. Key bırakılınca: `settingPress=false`, blok kalkar
6. Sonraki basışlarda aim zaten hedede → blok yok, sol+hook anında çalışır

**İlgili Dosyalar:**
- `src/game/client/components/controls.h` → state değişkenleri
- `src/game/client/components/controls.cpp` → `ConKeyAimAngle`, `ConKeyAimAngle2`, `OnRender`, `SnapInput`

**Bind Kullanımı:**
```
bc_aim_angle -214
bc_aim_angle2 -590
bc_aim_angle_speed 25
bind 2 "+left; +hook; +aim_angle"
bind 1 "+right; +hook; +aim_angle2"
```

---

### 2. Discord RPC

**Amaç:** Discord'da BestClient aktivitesi gösterir.

**App ID:** `1444576875774083133` (BestClient Discord uygulaması)

**Config:**
```
tc_discord_rpc 1    // 0=kapalı, 1=açık (yeniden başlatma gerektirir)
```

**UI Yeri:** Settings → TClient → Info → "Integration" → Enable Discord RPC

**SDK Konumu:** `ddnet-libs/discord/linux/lib64/discord_game_sdk.so`

**Build Seçenekleri:**
- `DISCORD=ON` → Discord SDK ile derle
- `DISCORD_DYNAMIC=ON` → SDK'yı çalışma zamanında yükle (dlopen)

**Not:** Discord aynı anda sadece bir oyun aktivitesi gösterir. BestClient açıkken başka bir uygulamanın RPC'si aktifse görünmeyebilir.

---

## 🆕 Yeni Özellik Ekleme

### 1. Yeni Config Değişkeni
```cpp
// src/engine/shared/config_variables_bestclient.h içine ekle:
MACRO_CONFIG_INT(BcYeniAyar, bc_yeni_ayar, 0, 0, 100, CFGFLAG_CLIENT | CFGFLAG_SAVE, "Açıklama")
MACRO_CONFIG_STR(BcYeniText, bc_yeni_text, 64, "varsayilan", CFGFLAG_CLIENT | CFGFLAG_SAVE, "Açıklama")
```
Erişim: `g_Config.m_BcYeniAyar`

### 2. Yeni Console Komutu
```cpp
// controls.h içine:
static void ConYeniKomut(IConsole::IResult *pResult, void *pUserData);

// controls.cpp OnConsoleInit() içine:
Console()->Register("yeni_komut", "?i", CFGFLAG_CLIENT, ConYeniKomut, this, "Açıklama");

// Implementasyon:
void CControls::ConYeniKomut(IConsole::IResult *pResult, void *pUserData) { ... }
```

### 3. Yeni Bind (+komut)
```cpp
Console()->Register("+yeni_bind", "", CFGFLAG_CLIENT, ConKeyYeniBind, this, "Açıklama");
// Handler pResult->GetInteger(0) → 1=basıldı, 0=bırakıldı
```

### 4. UI Menü Elemanı
```cpp
// menus_bestclient.cpp veya menus_tclient.cpp içine:
DoButton_CheckBoxAutoVMarginAndSet(&g_Config.m_BcYeniAyar, "Label", &g_Config.m_BcYeniAyar, &View, LineSize);
// Veya scrollbar:
Ui()->DoScrollbarOption(&g_Config.m_BcYeniAyar, &g_Config.m_BcYeniAyar, &Button, "Label", 0, 100);
```

---

## 🔄 Yeni Sürüme Güncelleme

BestClient ana branch'i güncellendiğinde:

```bash
# Upstream ekle (bir kere yapılır)
git remote add upstream https://github.com/SollyBunny/TaterClient-ddnet.git  # veya BestClient'ın gerçek upstream'i

# Güncellemeleri al
git fetch upstream
git merge upstream/main

# Çakışmaları çöz (controls.cpp, config dosyaları değişmiş olabilir)
# Sonra build et
cmake --build build --target DDNet -j$(nproc)
```

**Dikkat edilecekler:**
- `config_variables_bestclient.h`: Yeni eklediğimiz değişkenler korunmalı
- `controls.cpp`: Aim angle blokları `SnapInput`'ta `ResolveMovementDirection`'dan **sonra** olmalı
- `discord.cpp`: App ID (`1444576875774083133`) değiştirilmemeli
- `ddnet-libs/`: `.gitignore`'da olduğundan Discord SDK'yı tekrar kopyalamak gerekebilir

---

## 🐛 Bilinen Davranışlar

| Durum | Neden | Çözüm |
|-------|-------|-------|
| Aim -213 gösteriyor | double yerine float kullan | `std::cos/sin` ile `float` cast |
| Mouse spam'de kilitlenme | İki aim sistemi çakışıyor | Her bind diğerini iptal eder (mutual cancel) |
| Discord görünmüyor | Başka uygulama RPC'yi alıyor | Diğer uygulamayı kapat |
| Hook erken atıyor | `settingPress` kontrolü | Key bırakılana kadar blok devam eder |

---

## 📝 Commit Formatı

```
feat: yeni özellik açıklaması
fix: düzeltme açıklaması
refactor: yeniden yapılandırma
```

**Repository:** https://github.com/prayjofir/BestClientangle
