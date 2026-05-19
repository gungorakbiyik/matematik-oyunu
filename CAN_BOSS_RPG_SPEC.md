# Can Boss RPG — Teknik Spec Dokümanı
> Claude Code için tam uygulama rehberi. Bu dokümanı oku ve oyunu sıfırdan yaz.

---

## 1. GENEL BAKIŞ

**Tür:** Matematik eğitim RPG oyunu  
**Hedef kitle:** Ortaokul öğrencileri (11–14 yaş), Türkçe arayüz  
**Platform:** Tek HTML dosyası (veya HTML + CSS + JS ayrı dosyalar), GitHub Pages'de yayınlanacak  
**Çözünürlük:** Masaüstü öncelikli, mobil de desteklenecek (responsive)

---

## 2. DOSYA YAPISI

```
can-boss-rpg/
├── index.html          ← Ana HTML iskelet, tüm ekranları içerir
├── style.css           ← Tüm CSS (CSS variables, ekranlar, animasyonlar)
├── js/
│   ├── main.js         ← Başlangıç noktası, state yönetimi, ekran geçişleri
│   ├── map.js          ← Harita: WASD hareketi, joystick, proximity detection
│   ├── questions.js    ← Soru havuzu + soru sistemi (ısınma & boss)
│   ├── battle.js       ← Boss savaş mekaniği, HP sistemi
│   ├── shop.js         ← Mağaza: satın alma, coin sistemi
│   └── leaderboard.js  ← localStorage liderlik tablosu
├── data/
│   └── questions.json  ← Soru havuzu (opsiyonel, JS içine de alınabilir)
└── assets/
    └── (ses dosyaları, varsa)
```

> **Not:** Eğer tek dosya istiyorsan her şeyi `index.html` içine koy,
> `<style>` ve `<script>` tagları içinde. Daha kolay deploy edilir.

---

## 3. RENK PALETİ & FONT

```css
:root {
  --orange:      #ff6200;
  --orange-glow: rgba(255, 98, 0, 0.55);
  --pink:        #ff2d78;
  --pink-glow:   rgba(255, 45, 120, 0.5);
  --yellow:      #ffc800;
  --teal:        #00dfc8;
  --green:       #00c853;
  --red:         #ff3040;
  --bg:          #08091a;
  --panel:       #0d1128;
  --border:      #1e2540;
}

/* Font: Google Fonts'tan yükle */
@import url('https://fonts.googleapis.com/css2?family=Press+Start+2P&display=swap');

body {
  font-family: 'Press Start 2P', monospace;
  background: var(--bg);
  color: #ddd8cc;
}
```

---

## 4. EKRAN AKIŞI

```
[ANA MENÜ]
    │  isim gir + BAŞLA butonuna bas
    ▼
[SINIF SEÇİMİ]
    │  Şövalye / Büyücü / Okçu seç + HARITAYA GEÇ
    ▼
[HARİTA]  ←──────────────────────────────────┐
    │  WASD ile yürü, E ile gir               │
    ├──► [ISINMA overlay]  ── ÇIK ────────────┤
    ├──► [MAĞAZA overlay]  ── ÇIK ────────────┤
    └──► [BOSS SAVAŞI overlay]                │
              │ kazan → level++ ──────────────┘
              │ level 3'ü kazan
              ▼
         [SONUÇ EKRANI]
              │ ANA MENÜ butonu
              ▼
         [ANA MENÜ]  (liderlik tablosu güncellendi)
```

---

## 5. EKRANLAR

### 5.1 Ana Menü (`#screen-menu`)

**Görsel:** Koyu arka plan, turuncu neon çerçeveli kutu, köşelerde altın ◆ simgesi  
**Elemanlar:**
- Oyun başlığı: "CAN BOSS RPG" (3 satır, büyük pixel font, turuncu neon glow)
- Input alanı: placeholder "İSMİN?" — turuncu kenarlı, siyah bg
- BAŞLA butonu: turuncu bg, siyah metin
- Divider: "✦ EN İYİ OYUNCULAR ✦"
- Liderlik tablosu: #, İSİM, SINIF, LV, PUAN kolonları

**Davranış:**
- Enter tuşu da BAŞLA'yı tetikler
- İsim boşsa input'a focus ver, geçme
- Liderlik tablosu localStorage'dan okunur

---

### 5.2 Sınıf Seçimi (`#screen-class`)

**Görsel:** 3 kart yan yana, her sınıfın rengi farklı

**3 Sınıf:**

| Sınıf | Renk | CAN | SALDIRIM | SAVUNMA | HIZ |
|-------|------|-----|----------|---------|-----|
| Şövalye | `#6699dd` | 9 | 6 | 10 | 4 |
| Büyücü | `#cc88ff` | 5 | 10 | 3 | 7 |
| Okçu | `#44cc88` | 7 | 8 | 5 | 10 |

**Her kartta:**
- Pixel art sprite SVG (bkz. Bölüm 9)
- Sınıf adı
- Kısa açıklama
- 4 stat barı: CAN (yeşil), SALDIRIM (turuncu), SAVUNMA (mavi), HIZ (sarı)
  - Her bar 10 segment, dolu segmentler renkli, boşlar `#1a2038`
- Özel yetenek kutusu:
  - Şövalye → "🛡️ DEMİR KALE": Her 5 soruda 1 kez, 1 tur sıfır hasar
  - Büyücü → "💥 BÜYÜLÜ ATILIŞ": Her 4 soruda 1 kez, 2x hasar
  - Okçu → "⚡ HIZLI ATIŞ": Her soruda aktif, 5 sn içinde cevap = +%50 hasar
- Cooldown göstergesi: ● ● ● ● ○ şeklinde daireler

**Seçim davranışı:**
- Karta tıklayınca o kart seçili duruma geçer (renkli border + glow)
- Seçili kartın sağ üstünde "✓" göster
- "HARITAYA GEÇ ▶" butonu: sınıf seçilince aktif olur

**HP Hesabı (sınıfa göre):**
```javascript
const baseHp = 100;
const playerMaxHp = baseHp + (classStats.hp * 5);
// Şövalye: 100 + 45 = 145 HP
// Büyücü:  100 + 25 = 125 HP
// Okçu:    100 + 35 = 135 HP
```

---

### 5.3 Harita (`#screen-map`)

**Üst HUD (her zaman görünür):**
```
[● MENÜ]                              [● 15] [♥ 100/145] [Lv 1]
İsim: AHMET                     coin    hp       level
Sınıf: ŞÖVALYE
```

**Harita alanı (620×460 px, mobilde tam genişlik):**
- Koyu mavi-gri zemin: `#151c30`
- Taş zemin çizgisi: 32×32 grid, `rgba(0,0,0,0.22)` çizgiler
- Kenar karartma (vignette): `inset 0 0 80px rgba(8,9,26,0.8)`
- Turuncu neon border

**4 meşale (köşelerde):**
- Titreyen alev CSS animasyonu (bkz. Bölüm 8.1)
- Turuncu radial gradient halo

**3 lokasyon:**

| Lokasyon | Pozisyon | Renk | Icon |
|----------|----------|------|------|
| ISINMA | Sol üst (x:87, y:80) | `#00dfc8` (teal) | Matematik kitabı SVG |
| MAĞAZA | Sağ üst (x:533, y:80) | `#ffc800` (sarı) | Altın sandık SVG |
| BOSS KAPISI | Alt orta (x:310, y:393) | `#ff2d78` (pembe) | Kuru kafa kapı SVG |

**Karakter (pixel art SVG):**
- Boyut: 34×46 px ekranda, `viewBox="0 0 26 36"`
- Başlangıç pozisyonu: x=310, y=230 (merkez)
- Animasyon: `idle-bob` — yukarı aşağı 4px, 1.3s ease-in-out infinite

**Hareket:**
```javascript
// WASD + Arrow keys
const SPEED = 2.6; // px/frame
const MAP_W = 620, MAP_H = 460;

document.addEventListener('keydown', e => keys[e.key] = true);
document.addEventListener('keyup',   e => keys[e.key] = false);

function gameLoop() {
  let dx = 0, dy = 0;
  if (keys['w'] || keys['W'] || keys['ArrowUp'])    dy = -1;
  if (keys['s'] || keys['S'] || keys['ArrowDown'])  dy =  1;
  if (keys['a'] || keys['A'] || keys['ArrowLeft'])  dx = -1;
  if (keys['d'] || keys['D'] || keys['ArrowRight']) dx =  1;

  // Joystick override (mobil)
  if (joystick.active) { dx = joystick.x; dy = joystick.y; }

  if (dx !== 0 || dy !== 0) {
    const len = Math.sqrt(dx*dx + dy*dy);
    pos.x = Math.max(28, Math.min(MAP_W - 28, pos.x + dx/len * SPEED));
    pos.y = Math.max(28, Math.min(MAP_H - 28, pos.y + dy/len * SPEED));
  }

  updateCharPosition();
  checkProximity();
  requestAnimationFrame(gameLoop);
}
```

**Proximity Detection (E tuşu):**
```javascript
const PROXIMITY_THRESHOLD = 72; // piksel

const LOCATIONS = [
  { id: 'loc-warm', cx: 87,  cy: 80,  action: openWarmup,  label: 'ISINMA'      },
  { id: 'loc-shop', cx: 533, cy: 80,  action: openShop,    label: 'MAĞAZA'      },
  { id: 'loc-boss', cx: 310, cy: 393, action: openBoss,    label: 'BOSS KAPISI' },
];

function checkProximity() {
  let nearest = null;
  for (const loc of LOCATIONS) {
    const dist = Math.hypot(pos.x - loc.cx, pos.y - loc.cy);
    if (dist < PROXIMITY_THRESHOLD) { nearest = loc; break; }
  }

  if (nearest !== currentNear) {
    // Önceki lokasyonun near efektini kaldır
    if (currentNear) removeNearEffect(currentNear.id);
    // Yeni lokasyona near efekti ekle + E prompt göster
    if (nearest) {
      addNearEffect(nearest.id);
      showEPrompt(nearest.label);
    } else {
      hideEPrompt();
    }
    currentNear = nearest;
  }
}

document.addEventListener('keydown', e => {
  if ((e.key === 'e' || e.key === 'E') && currentNear) {
    currentNear.action();
  }
});
```

**E Prompt:**
```html
<!-- Karakterin üstünde, absolute pozisyonlu -->
<div id="e-prompt" style="display:none;">
  <div class="e-key">E</div>          <!-- sarı kare, bounce animasyonu -->
  <div class="e-text">ISINMA</div>    <!-- 5px beyaz metin -->
</div>
```

---

### 5.4 Isınma Overlay (`#overlay-warmup`)

**Konum:** Harita alanının ortasında, yarı saydam siyah backdrop  
**Border:** 3px solid `#00dfc8`, box-shadow teal glow  
**Sağ üst:** "✕ ÇIK" butonu — her zaman görünür, kapanınca haritaya döner

**İçerik:**
```
📖 ISINMA          1 / 4        [✕ ÇIK]
─────────────────────────────────────────
        3 + 5 × 2 = ?
─────────────────────────────────────────
[A]  16          [B]  13
[C]  11          [D]  10
─────────────────────────────────────────
         (geri bildirim alanı)
```

**Davranış:**
1. 4 soru rastgele havuzdan seçilir, karıştırılır
2. Cevap seçilince:
   - **Doğru:** Buton yeşil, "✓ DOĞRU! +5 altın", yeşil flash, +5 coin, +10 puan
   - **Yanlış:** Seçilen buton kırmızı, doğru buton yeşil, "✗ YANLIŞ! -10 HP", kırmızı flash + titreme
3. 1.1 saniye sonra sonraki soruya geç
4. 4 soru bitince overlay kapanır, haritaya dön
5. Klavye: A/B/C/D tuşları da çalışır

---

### 5.5 Mağaza Overlay (`#overlay-shop`)

**Border:** 3px solid `#ffc800`, sarı glow  
**Sağ üst:** "✕ ÇIK"

**İçerik:**
```
🧰 MAĞAZA                       [✕ ÇIK]
           ● 15 Altın
────────────────────────────────────────
💉 | +10 MAX HP          | [● 20]
   | Mevcut MAX HP: 145  |
────────────────────────────────────────
💡 | İPUCU (1 ADET)      | [● 10]
   | Sahip olunan: 0     |
────────────────────────────────────────
```

**Davranış:**
- Yeterli coin yoksa satın alma butonu disabled
- HP satın alınca MAX HP +10 artır, mevcut HP de +10 (maxHp'yi aşmaz)
- İpucu boss savaşında bir şıkkı elemine eder
- Satın alma sonrası coin ve açıklamalar güncellenir

---

### 5.6 Boss Savaşı Overlay (`#overlay-boss`)

**Border:** 3px solid `#ff2d78`, pembe glow  
**Sağ üst:** "✕ ÇIK" (her zaman görünür, savaştan çıkılabilir)

**Üst bant (levele göre boss):**
```
💀 BOSS SAVAŞI — LV 1
```

**Savaş alanı:**
```
👤 KAHRAMAN              🪨 TAŞ GOLEM
[██████████] 145/145     [█████████████] 80/80

   [Şövalye sprite]  VS  [Boss sprite]

(Dönen rün halkaları + kıvılcım partikülleri)
```

**Soru alanı:**
```
─────────────────────────────────────────
        (12 + 8) ÷ 4 × 3 = ?
─────────────────────────────────────────
[A]  9          [B]  6
[C]  15         [D]  20
─────────────────────────────────────────
(geri bildirim)
```

**Davranış:**
- **Doğru cevap:**
  - Boss HP azalır: `damage = classStats.atk * 2 + 10`
  - Şövalye özel: her 5 soruda 1 kez → damage 0 alır (DEMİR KALE aktif değil, verir)
  - Büyücü özel: her 4 soruda 1 kez → damage × 2 (BÜYÜLÜ ATILIŞ)
  - Okçu özel: 5 saniye içinde cevap → damage × 1.5 (HIZLI ATIŞ)
  - Yeşil flash, boss sarsılır, kırmızı damage sayısı uçar
- **Yanlış cevap:**
  - Oyuncu HP azalır: `damage = boss.dmg` (levele göre)
  - Kırmızı flash, ekran titrer
- **Boss HP = 0:** Kazanıldı → `bossWin()`
- **Oyuncu HP = 0:** Kaybedildi → `bossLose()`

---

### 5.7 Sonuç Ekranı

```
🏆 KAZANDIN!    (veya)   💀 KAYBETTİN

İsim:  AHMET
Sınıf: ŞÖVALYE
Seviye: 3
Puan: ● 250

[ANA MENÜ ▶]
```

- Puan localStorage'a kaydedilir
- ANA MENÜ'ye dönünce liderlik tablosu güncellenir

---

## 6. BOSS SİSTEMİ

| Level | Boss Adı | Max HP | Hasar/soru |
|-------|----------|--------|-----------|
| 1 | 🪨 Taş Golem | 80 | 12 |
| 2 | ❄️ Bfröst | 110 | 16 |
| 3 | 👿 İblis Lordu | 150 | 22 |
| 4+ | (tekrar veya sonsuz mod) | artar | artar |

**Level atlama:**
```javascript
function bossWin() {
  score += 50;
  coins += 20;
  level++;
  if (level > MAX_LEVEL) {
    endGame(true); // tüm bossları yendi
  } else {
    // haritaya dön, yeni levelde devam et
    showMessage('Lv ' + level + '\'e yükseldin! +20 altın');
    returnToMap();
  }
}
```

---

## 7. SORU SİSTEMİ

### 7.1 Soru Formatı
```javascript
const questions = [
  {
    q: '3 + 5 × 2 = ?',
    opts: ['16', '13', '11', '10'],  // A, B, C, D
    ans: 1,                           // doğru cevap index'i (0-3)
    difficulty: 1                     // 1=kolay, 2=orta, 3=zor
  },
  // ...
];
```

### 7.2 Minimum Soru Havuzu (40 soru önerilir)

```javascript
// KOLAY (difficulty: 1) — Level 1
{ q:'3 + 5 × 2 = ?',          opts:['16','13','11','10'],   ans:1, diff:1 },
{ q:'(12+8) ÷ 4 × 3 = ?',     opts:['9','6','15','20'],     ans:2, diff:1 },
{ q:'7² − 25 = ?',             opts:['49','24','25','15'],   ans:1, diff:1 },
{ q:'4 × (6+3) ÷ 6 = ?',      opts:['8','9','4','6'],       ans:3, diff:1 },
{ q:'√144 + 8 = ?',            opts:['12','14','22','20'],   ans:3, diff:1 },
{ q:'15 × 4 ÷ 12 = ?',        opts:['5','6','4','8'],       ans:0, diff:1 },
{ q:'2³ + 3² = ?',             opts:['13','15','17','19'],   ans:2, diff:1 },
{ q:'(20−5) × 3 ÷ 9 = ?',     opts:['15','10','3','5'],     ans:3, diff:1 },
{ q:'48 ÷ 6 + 7 × 2 = ?',     opts:['21','16','20','22'],   ans:3, diff:1 },
{ q:'(7+3) × (4−2) = ?',      opts:['14','16','20','24'],   ans:2, diff:1 },

// ORTA (difficulty: 2) — Level 2
{ q:'x + 15 = 32, x = ?',     opts:['47','17','27','12'],   ans:1, diff:2 },
{ q:'3/4 + 1/2 = ?',          opts:['4/6','5/4','1¼','1½'],  ans:2, diff:2 },
{ q:'%25 of 80 = ?',          opts:['25','20','15','30'],   ans:1, diff:2 },
{ q:'5² × 4 − 10 = ?',        opts:['90','110','100','80'], ans:0, diff:2 },
{ q:'144 ÷ 12 × 5 = ?',       opts:['60','72','65','70'],   ans:0, diff:2 },
{ q:'2 × (3+4)² − 1 = ?',     opts:['97','95','99','100'],  ans:0, diff:2 },
{ q:'√(9 × 16) = ?',          opts:['25','12','15','18'],   ans:1, diff:2 },
{ q:'4x = 36, x = ?',         opts:['9','8','7','6'],       ans:0, diff:2 },
{ q:'1/3 × 90 + 5 = ?',       opts:['35','30','40','25'],   ans:0, diff:2 },
{ q:'(2+3)³ ÷ 25 = ?',        opts:['5','4','3','6'],       ans:0, diff:2 },

// ZOR (difficulty: 3) — Level 3
{ q:'3x + 7 = 22, x = ?',     opts:['4','5','6','3'],       ans:1, diff:3 },
{ q:'(a+b)² = a²+2ab+b². (3+4)² = ?', opts:['47','49','50','51'], ans:1, diff:3 },
{ q:'%30 artış → 100\'de kaç?',opts:['110','120','125','130'],ans:3, diff:3 },
{ q:'2^10 = ?',                opts:['512','1024','2048','256'],ans:1, diff:3 },
{ q:'n! = 120, n = ?',         opts:['4','5','6','7'],       ans:1, diff:3 },
```

### 7.3 Soru Seçimi
```javascript
function getQuestionsForLevel(level, count) {
  const diffMap = { 1: [1, 2], 2: [1, 2, 3], 3: [2, 3] };
  const allowed = diffMap[Math.min(level, 3)] || [1, 2, 3];
  const pool = questions.filter(q => allowed.includes(q.diff));
  return shuffle(pool).slice(0, count);
}

function shuffle(arr) {
  return [...arr].sort(() => Math.random() - 0.5);
}
```

---

## 8. ANİMASYONLAR & EFEKTLER

### 8.1 Meşale Animasyonu
```css
.torch-flame {
  width: 11px; height: 18px;
  background: radial-gradient(ellipse at 50% 85%,
    #ff6200 0%, #ffc800 40%, #ff2d00 75%, transparent 100%);
  border-radius: 50% 50% 30% 30%;
  transform-origin: bottom;
  filter: blur(1px);
  animation: flicker 0.14s ease-in-out infinite alternate;
}
@keyframes flicker {
  from { transform: skewX(-6deg) scaleY(0.93); }
  to   { transform: skewX(6deg) scaleY(1.07); }
}

.torch-glow {
  width: 60px; height: 60px;
  background: radial-gradient(circle, rgba(255,140,0,0.22) 0%, transparent 70%);
  border-radius: 50%;
  animation: flicker-glow 0.25s ease-in-out infinite alternate;
}
@keyframes flicker-glow {
  from { opacity: 0.4; transform: scale(0.88); }
  to   { opacity: 0.9; transform: scale(1.12); }
}
```

### 8.2 Ekran Flash
```javascript
function doFlash(color) {
  const flash = document.getElementById('flash-overlay');
  flash.style.background = color;
  flash.style.animation = 'none';
  void flash.offsetWidth; // reflow trick
  flash.style.animation = 'flash-anim 0.4s ease-out forwards';
}
// Doğru cevap: doFlash('rgba(0,200,83,0.28)')
// Yanlış cevap: doFlash('rgba(255,48,64,0.38)')
```

```css
#flash-overlay {
  position: fixed; inset: 0;
  pointer-events: none; z-index: 9999; opacity: 0;
}
@keyframes flash-anim { 0% { opacity: 1; } 100% { opacity: 0; } }
```

### 8.3 Ekran Titremesi
```css
@keyframes shake {
  0%,100% { transform: translateX(0); }
  15%     { transform: translateX(-9px); }
  30%     { transform: translateX(9px); }
  45%     { transform: translateX(-6px); }
  60%     { transform: translateX(6px); }
  75%     { transform: translateX(-3px); }
  90%     { transform: translateX(3px); }
}
/* Kullanım: el.style.animation = 'shake 0.4s ease-out'; */
```

### 8.4 Uçan Hasar Sayısı
```javascript
function showDamageNumber(text, x, y, color) {
  const el = document.createElement('div');
  el.style.cssText = `
    position: absolute; left: ${x}px; top: ${y}px;
    font-family: 'Press Start 2P', monospace;
    font-size: 18px; color: ${color};
    pointer-events: none; z-index: 30;
    animation: dmg-float 1s ease-out forwards;
  `;
  el.textContent = text;
  document.getElementById('battle-area').appendChild(el);
  setTimeout(() => el.remove(), 1000);
}
@keyframes dmg-float {
  0%   { opacity:1; transform: translateY(0) scale(1); }
  60%  { opacity:1; transform: translateY(-40px) scale(1.3); }
  100% { opacity:0; transform: translateY(-70px) scale(0.7); }
}
```

### 8.5 Karakter Idle Bob
```css
@keyframes idle-bob {
  0%,100% { transform: translate(-50%, -100%) translateY(0); }
  50%     { transform: translate(-50%, -100%) translateY(-4px); }
}
#character {
  position: absolute;
  transform: translate(-50%, -100%);
  animation: idle-bob 1.3s ease-in-out infinite;
}
```

---

## 9. SPRITE'LAR (SVG)

### 9.1 Oyuncu — Şövalye (tüm sınıflar için varsayılan harita)
```html
<!-- viewBox="0 0 26 36", image-rendering: pixelated -->
<svg width="34" height="46" viewBox="0 0 26 36" style="image-rendering:pixelated">
  <!-- Kask -->
  <rect x="8" y="0" width="10" height="2" fill="#5f7a99"/>
  <rect x="7" y="2" width="12" height="6" fill="#7090b0"/>
  <rect x="9" y="3" width="8" height="4" fill="#2a3a4a"/><!-- vizör -->
  <rect x="10" y="4" width="6" height="2" fill="#3a4e60"/>
  <rect x="7" y="6" width="2" height="3" fill="#5f7a99"/>
  <rect x="17" y="6" width="2" height="3" fill="#5f7a99"/>
  <!-- Boyun -->
  <rect x="10" y="8" width="6" height="2" fill="#bb8866"/>
  <!-- Omuz zırhı -->
  <rect x="4" y="10" width="5" height="4" fill="#5f7a99"/>
  <rect x="17" y="10" width="5" height="4" fill="#5f7a99"/>
  <!-- Göğüs zırhı -->
  <rect x="8" y="10" width="10" height="8" fill="#3a5599"/>
  <rect x="12" y="11" width="2" height="6" fill="#6688cc"/><!-- haç -->
  <rect x="10" y="13" width="6" height="2" fill="#6688cc"/>
  <!-- Kemer -->
  <rect x="8" y="18" width="10" height="2" fill="#7a5522"/>
  <!-- Kalkan (sol) -->
  <rect x="4" y="14" width="4" height="6" fill="#5f7a99"/>
  <rect x="1" y="12" width="5" height="11" fill="#2244aa"/>
  <rect x="2" y="13" width="3" height="9" fill="#3355cc"/>
  <rect x="3" y="17" width="1" height="2" fill="#ffcc00"/><!-- amblemi -->
  <!-- Kılıç (sağ) -->
  <rect x="18" y="14" width="4" height="5" fill="#5f7a99"/>
  <rect x="22" y="8" width="2" height="16" fill="#aabbc8"/><!-- bıçak -->
  <rect x="20" y="15" width="5" height="2" fill="#7a5522"/><!-- koruyucu -->
  <!-- Bacaklar -->
  <rect x="9" y="20" width="4" height="8" fill="#2c4488"/>
  <rect x="13" y="20" width="4" height="8" fill="#2c4488"/>
  <!-- Çizmeler -->
  <rect x="8" y="28" width="5" height="4" fill="#263344"/>
  <rect x="13" y="28" width="5" height="4" fill="#263344"/>
</svg>
```

### 9.2 Taş Golem (Boss Lv 1)
```html
<svg width="100" height="120" viewBox="0 0 36 42" style="image-rendering:pixelated">
  <!-- Büyük kaya kafa -->
  <rect x="8" y="3" width="20" height="17" fill="#3e3828" rx="2"/>
  <!-- Çatlak çizgiler -->
  <line x1="10" y1="7" x2="17" y2="5" stroke="#2e2918" stroke-width="1.2"/>
  <line x1="19" y1="9" x2="26" y2="7" stroke="#2e2918" stroke-width="1.2"/>
  <!-- Lav gözler -->
  <rect x="11" y="8" width="5" height="5" fill="#ff5500">
    <animate attributeName="opacity" values=".55;1;.55" dur=".9s" repeatCount="indefinite"/>
  </rect>
  <rect x="20" y="8" width="5" height="5" fill="#ff5500">
    <animate attributeName="opacity" values=".55;1;.55" dur=".9s" repeatCount="indefinite" begin=".18s"/>
  </rect>
  <!-- İçten parlayan göz -->
  <rect x="12" y="9" width="3" height="3" fill="#ff9900">
    <animate attributeName="opacity" values=".4;1;.4" dur=".55s" repeatCount="indefinite"/>
  </rect>
  <rect x="21" y="9" width="3" height="3" fill="#ff9900">
    <animate attributeName="opacity" values=".4;1;.4" dur=".55s" repeatCount="indefinite" begin=".12s"/>
  </rect>
  <!-- Çatlak ağız -->
  <path d="M12 17 L14 15 L16 17 L18 15 L20 17 L22 15 L24 17" stroke="#ff5500" stroke-width="1.1" fill="none" opacity=".75"/>
  <!-- Masif vücut -->
  <rect x="5" y="20" width="26" height="17" fill="#363020" rx="1"/>
  <!-- Vücut lav çatlakları -->
  <path d="M9 22 L11 28 L8 32" stroke="#ff5500" stroke-width="1.1" fill="none" opacity=".55"/>
  <path d="M22 24 L25 30 L27 32" stroke="#ff5500" stroke-width="1" fill="none" opacity=".5"/>
  <!-- Dev kollar -->
  <rect x="0" y="20" width="7" height="15" fill="#363020"/>
  <rect x="29" y="20" width="7" height="15" fill="#363020"/>
  <!-- Yumruklar -->
  <rect x="0" y="33" width="9" height="8" fill="#2e2918" rx="1"/>
  <rect x="27" y="33" width="9" height="8" fill="#2e2918" rx="1"/>
  <!-- Bacaklar -->
  <rect x="8" y="37" width="8" height="5" fill="#242010"/>
  <rect x="20" y="37" width="8" height="5" fill="#242010"/>
</svg>
```

### 9.3 Bfröst / Buz Büyücüsü (Boss Lv 2)
```html
<svg width="100" height="120" viewBox="0 0 36 42" style="image-rendering:pixelated">
  <!-- Sivri şapka -->
  <polygon points="18,0 10,13 26,13" fill="#003388"/>
  <polygon points="18,0 12,11 24,11" fill="#0044bb"/>
  <!-- Buz kristalleri -->
  <polygon points="18,0 16,5 20,5" fill="#88ddff"/>
  <polygon points="12,7 10,12 14,12" fill="#88ddff" opacity=".6"/>
  <polygon points="24,7 22,12 26,12" fill="#88ddff" opacity=".6"/>
  <!-- Şapka kenarı -->
  <rect x="7" y="13" width="22" height="3" fill="#002277"/>
  <!-- Buz mavisi yüz -->
  <rect x="11" y="16" width="14" height="11" fill="#99ccee"/>
  <!-- Gözler -->
  <rect x="13" y="18" width="3" height="3" fill="#0088ff"/>
  <rect x="20" y="18" width="3" height="3" fill="#0088ff"/>
  <!-- Donmuş ağız -->
  <rect x="14" y="24" width="8" height="2" fill="#0055aa"/>
  <!-- Uzun cüppe -->
  <rect x="9" y="27" width="18" height="14" fill="#001155"/>
  <rect x="9" y="27" width="18" height="2" fill="#003388"/>
  <!-- Kollar -->
  <rect x="3" y="27" width="7" height="11" fill="#002277"/>
  <rect x="26" y="27" width="7" height="11" fill="#002277"/>
  <!-- Asa -->
  <rect x="0" y="10" width="2" height="28" fill="#1a3366"/>
  <polygon points="1,10 -2,6 4,6" fill="#00aaff"/>
  <!-- Buz kristali parlaması -->
  <circle cx="1" cy="8" r="3" fill="none" stroke="#00ccff" stroke-width=".7">
    <animate attributeName="r" values="2;5;2" dur="1.6s" repeatCount="indefinite"/>
    <animate attributeName="opacity" values=".5;0;.5" dur="1.6s" repeatCount="indefinite"/>
  </circle>
  <!-- Yüzen buz partikülleri -->
  <circle cx="29" cy="22" r="1.5" fill="#55ddff">
    <animate attributeName="cy" values="22;16;22" dur="2s" repeatCount="indefinite"/>
    <animate attributeName="opacity" values=".3;.9;.3" dur="2s" repeatCount="indefinite"/>
  </circle>
</svg>
```

### 9.4 İblis Lordu (Boss Lv 3)
```html
<svg width="108" height="130" viewBox="0 0 36 42" style="image-rendering:pixelated">
  <!-- Kanatlar -->
  <polygon points="0,10 8,26 14,19" fill="#3a0018" opacity=".9"/>
  <polygon points="36,10 28,26 22,19" fill="#3a0018" opacity=".9"/>
  <!-- Boynuzlar -->
  <polygon points="10,0 12,11 7,11" fill="#bb1a00"/>
  <polygon points="26,0 24,11 29,11" fill="#bb1a00"/>
  <polygon points="10,0 11,4 9,4" fill="#ff3300"/>
  <polygon points="26,0 25,4 27,4" fill="#ff3300"/>
  <!-- Kafa -->
  <rect x="10" y="5" width="16" height="14" fill="#7a0000" rx="1"/>
  <!-- Kor kırmızı gözler -->
  <rect x="12" y="8" width="4" height="4" fill="#ff1a00"/>
  <rect x="20" y="8" width="4" height="4" fill="#ff1a00"/>
  <rect x="13" y="9" width="2" height="2" fill="#ff8800">
    <animate attributeName="opacity" values=".4;1;.4" dur=".65s" repeatCount="indefinite"/>
  </rect>
  <rect x="21" y="9" width="2" height="2" fill="#ff8800">
    <animate attributeName="opacity" values=".4;1;.4" dur=".65s" repeatCount="indefinite" begin=".12s"/>
  </rect>
  <!-- Dişli ağız -->
  <rect x="12" y="15" width="12" height="3" fill="#2e0000"/>
  <rect x="13" y="15" width="2" height="2" fill="#ddd"/>
  <rect x="16" y="15" width="2" height="2" fill="#ddd"/>
  <rect x="19" y="15" width="2" height="2" fill="#ddd"/>
  <rect x="22" y="15" width="2" height="2" fill="#ddd"/>
  <!-- Boyun -->
  <rect x="14" y="19" width="8" height="3" fill="#600000"/>
  <!-- Karanlık zırh -->
  <rect x="8" y="22" width="20" height="13" fill="#1e000d"/>
  <rect x="10" y="23" width="16" height="11" fill="#2c0016"/>
  <polygon points="18,23 15,29 21,29" fill="#ff2d78" opacity=".7"/><!-- göğüs amblemi -->
  <!-- Omuz sivrileri -->
  <polygon points="8,22 5,18 10,22" fill="#aa1a00"/>
  <polygon points="28,22 31,18 26,22" fill="#aa1a00"/>
  <!-- Kollar -->
  <rect x="3" y="23" width="6" height="9" fill="#7a0000"/>
  <rect x="27" y="23" width="6" height="9" fill="#7a0000"/>
  <!-- Pençeler -->
  <rect x="3" y="32" width="2" height="5" fill="#aa1a00"/>
  <rect x="5" y="32" width="2" height="5" fill="#aa1a00"/>
  <rect x="29" y="32" width="2" height="5" fill="#aa1a00"/>
  <rect x="27" y="32" width="2" height="5" fill="#aa1a00"/>
  <!-- Bacaklar -->
  <rect x="10" y="35" width="6" height="6" fill="#1e000d"/>
  <rect x="20" y="35" width="6" height="6" fill="#1e000d"/>
  <!-- Pembe aura halkası -->
  <circle cx="18" cy="21" r="18" fill="none" stroke="#ff2d78" stroke-width=".5" opacity=".25">
    <animate attributeName="r" values="16;22;16" dur="2.2s" repeatCount="indefinite"/>
    <animate attributeName="opacity" values=".1;.4;.1" dur="2.2s" repeatCount="indefinite"/>
  </circle>
</svg>
```

---

## 10. MOBİL DESTEK

### 10.1 Joystick
```javascript
const joystick = { active: false, x: 0, y: 0, origin: { x: 0, y: 0 } };
const MAX_JOY = 30;

const ring = document.getElementById('joy-ring');
const thumb = document.getElementById('joy-thumb');

ring.addEventListener('touchstart', e => {
  e.preventDefault();
  joystick.active = true;
  const r = ring.getBoundingClientRect();
  joystick.origin = { x: r.left + r.width/2, y: r.top + r.height/2 };
}, { passive: false });

document.addEventListener('touchmove', e => {
  if (!joystick.active) return;
  e.preventDefault();
  const t = e.touches[0];
  applyJoystick(t.clientX, t.clientY);
}, { passive: false });

document.addEventListener('touchend', () => {
  joystick.active = false;
  joystick.x = joystick.y = 0;
  thumb.style.transform = '';
});

function applyJoystick(cx, cy) {
  const dx = cx - joystick.origin.x;
  const dy = cy - joystick.origin.y;
  const dist = Math.min(Math.hypot(dx, dy), MAX_JOY);
  const angle = Math.atan2(dy, dx);
  thumb.style.transform = `translate(${Math.cos(angle)*dist}px, ${Math.sin(angle)*dist}px)`;
  joystick.x = dist > 6 ? Math.cos(angle) : 0;
  joystick.y = dist > 6 ? Math.sin(angle) : 0;
}
```

### 10.2 Responsive CSS
```css
/* Harita: masaüstünde merkez, mobilde tam ekran */
#dungeon {
  width: 620px;
  height: 460px;
}

@media (max-width: 680px) {
  #dungeon {
    width: 100vw !important;
    height: calc(100dvh - 42px) !important; /* HUD yüksekliği çıkarılır */
    border-left: none;
    border-right: none;
  }
  #joy-wrap { display: block; }   /* joystick göster */
  #e-button { display: flex; }    /* E butonu göster */
  .wasd-hint { display: none; }   /* WASD yazısını gizle */
}

/* Savaş butonları: mobilde büyük */
.answer-btn {
  min-height: 60px;   /* Apple HIG: min 44px, biz 60px yapıyoruz */
  font-size: 11px;
}
```

### 10.3 Mobil E Butonu
```html
<!-- Harita ekranının sağ alt köşesi -->
<button id="e-button" onclick="tryEnter()" style="display:none;">
  <span style="font-size:10px;">E</span>
  <span style="font-size:5px;">GİR</span>
</button>
```

---

## 11. LOCALStorage ŞEMASI

```javascript
const STORAGE_KEY = 'canbossrpg_leaderboard';

// Liderlik tablosuna kaydetme
function saveScore(entry) {
  const board = JSON.parse(localStorage.getItem(STORAGE_KEY) || '[]');
  board.push({
    name:  entry.name,     // "AHMET"
    cls:   entry.cls,      // "ŞÖVALYE"
    lv:    entry.level,    // 3
    score: entry.score,    // 250
    date:  Date.now()      // timestamp
  });
  board.sort((a, b) => b.score - a.score);
  localStorage.setItem(STORAGE_KEY, JSON.stringify(board.slice(0, 10))); // max 10
}

// Liderlik tablosunu okuma
function loadBoard() {
  return JSON.parse(localStorage.getItem(STORAGE_KEY) || '[]');
}
```

---

## 12. OYUN STATE OBJESİ

```javascript
const G = {
  // Oyuncu
  name: '',
  cls: null,          // { id, name, cc, stats, ab, ... }
  level: 1,
  coins: 15,
  hp: 100,
  maxHp: 100,
  hints: 0,
  score: 0,

  // Harita
  pos: { x: 310, y: 230 },
  keys: {},
  raf: null,
  nearLoc: null,

  // Soru sistemi
  qPool: [],
  qIdx: 0,
  warmupRemaining: 4,

  // Boss
  bossHp: 0,
  bossMaxHp: 0,
  bossDmg: 0,

  // Özel yetenek cooldown
  abilityCooldown: 0,   // kaç soruda bir sıfırlanacak
  abilityCount: 0,      // mevcut sayaç

  // Diğer
  answerLocked: false,
  joystick: { active: false, x: 0, y: 0 }
};
```

---

## 13. PUAN SİSTEMİ

| Olay | Puan |
|------|------|
| Isınma sorusu doğru | +10 |
| Boss sorusu doğru | +15 |
| Boss yenildi | +50 |
| Isınma altın kazanma | coin olarak |
| İpucu kullanma | -5 puan |

---

## 14. KOD YAZIM NOTLARI

1. **Tek HTML dosyası** tercih et — `<style>` ve `<script>` içinde her şey. GitHub Pages'de sorunsuz çalışır.
2. **requestAnimationFrame** loop haritada çalışır, overlay açıkken `cancelAnimationFrame` ile durdur.
3. **Overlay açık/kapalı** yönetimi: CSS `display:flex/none` ile, JS class toggle ile yap.
4. **Karakter pozisyonu** her frame'de CSS `left/top` olarak güncellenir (absolute positioned).
5. **Sprite SVG** inline HTML içinde olsun, `image-rendering: pixelated` unutma.
6. **Flash efekti** için `void el.offsetWidth` reflow trick'ini kullan (animasyon resetlenir).
7. **Mobil** için `touchstart` / `touchmove` / `touchend` event'leri `passive: false` ile dinle.
8. **LocalStorage** okuma/yazma işlemlerini `try/catch` içine al (private mode'da hata verebilir).

---

## 15. REFERANS DOSYALAR

Bu proje dizininde şunlar mevcut:

- **`Can Boss RPG - Prototype.html`** → Çalışan tam prototip (tüm sprite'lar, mekanikler, overlay'ler dahil). Claude Code bunu referans olarak okuyabilir.
- **`Can Boss RPG - Tasarım Önerileri.html`** → Görsel tasarım showcase'i (harita, savaş, sprite galeri, mobil).

> **Tavsiye:** Önce `Can Boss RPG - Prototype.html` dosyasını oku. Tüm çalışan kod orada.
> Bu spec onu nasıl temiz yazacağını anlatıyor.
