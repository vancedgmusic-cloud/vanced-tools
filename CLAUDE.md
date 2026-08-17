# Framework Brain Director

Dua single-file browser app yang berdiri sendiri, UI Bahasa Indonesia:

- `index.html` (~1.9k baris) — pipeline riset produk → storyboard → video prompt
  → ekspor props Remotion. Dokumen ini membahasnya kecuali disebut lain.
- `influencer.html` — AI Influencer Studio. Lihat bagian di bawah.

Keduanya tidak saling impor. Konvensinya sama (`getPath`/`setPath` + `data-path`,
`parseJSONLoose`, `postJSON`, key dibuang lewat pola `/key$/i`), jadi perbaikan
di salah satunya biasanya perlu disalin manual ke yang lain.

Satu file berisi `<style>` + markup + `<script>` (mulai baris ~302). `'use strict'`.
Tidak dipecah ke file terpisah — app dipakai lewat `file://` dan juga sebagai
Claude.ai Artifact (provider `builtin`).

## State

`state` = satu objek global di memori. **Tidak ada localStorage** — persistence
lewat ekspor/impor JSON (`saveProject()` / `loadProject(file)`). Artinya refresh =
kerja hilang. Kalau menambah field top-level ke `state`, daftarkan juga di array
whitelist di dalam `loadProject` (baris ~1870) atau field itu tidak akan ter-restore.

Akses nested pakai `getPath` / `setPath` dengan string path (`"remotion.harga"`),
dipasangkan ke DOM lewat atribut `data-path`.

## Alur

`STEPS` (baris ~358) = `setup → input → knowledge → marketing → angle → brief →
storyboard → prompts → caphash → export`. Tahap ber-`ai:true` dipanggil via
`promptStage(id, feedback)`. `REGEN_ORDER` menentukan tahap mana yang jadi kotor
(`state.dirty`) ketika tahap sebelumnya di-regenerate.

`F` (baris ~370) = definisi field per tahap. Sumber kebenaran untuk render form,
`kvText()` yang membangun konteks prompt, dan schema JSON (`schemaOf`). Tambah field
cukup di `F`.

## Provider AI

`callAI(prompt, maxTokens, images)` → `PROVIDERS`: `builtin` (Claude.ai Artifact,
tanpa key), `claude`, `gemini`, `groq`, `openrouter`, `custom` (OpenAI-compatible).

Dua mekanisme yang gampang terlewat kalau mengedit:

- **Loop kontinuasi** (`MAX_CHUNK=6`): kalau output kepotong batas token, potongan
  dikirim balik sebagai pesan assistant supaya AI melanjutkan. Berlaku di ketiga cabang.
- **Retry transien** (`postJSON`): 3 percobaan dengan backoff untuk 429/5xx dan pesan
  yang cocok regex `TRANSIENT`. Error non-transien dilempar langsung.

Output JSON diparse longgar: `parseJSONLoose` → `repairJSON` (menambal string/kurung
yang tidak tertutup). Jangan ganti dengan `JSON.parse` polos — model sering balas
JSON kepotong.

## Aturan domain (jangan dilanggar)

- **CTA hanya di scene terakhir.**
- `state.remotion` (harga, harga coret, diskon, toko, rating, jumlah review,
  kutipan review, nama reviewer) adalah **data faktual — AI tidak boleh mengarang**.
  `sanitizeRemotionProps(ai, base)` memaksa field ini kembali ke nilai dari `base`
  setelah AI memoles. Jangan longgarkan fungsi ini.
- `remotionMissing(props)` melaporkan field faktual yang masih kosong ke user,
  bukan mengisinya otomatis.
- Tema visual & gaya kamera yang sudah dipilih user bersifat FIXED — prompt builder
  menandainya eksplisit supaya AI tidak mengarang tema baru.

## Ekspor

- `exportStoryboard()` — storyboard teks/JSON
- `exportRemotion(useAI)` — `AffiliateAdProps` untuk komposisi `AffiliateAdVertical`.
  Render: `npx remotion render AffiliateAdVertical out/video.mp4 --props=file.json`

Untuk kerja Remotion, pakai skill `remotion-best-practices`.

## Keamanan

API key ada di `state.api`, memori saja. `saveProject()` menghapusnya dari file
ekspor berdasarkan **bentuk nama field**, bukan daftar nama:

```js
snap.api=Object.fromEntries(Object.entries(snap.api).map(([k,v])=>[k, /key$/i.test(k)?'':v]));
```

Artinya provider baru aman secara default — asal field kredensialnya diberi nama
berakhiran `Key` (`deepseekKey`, `mistralKey`, dst). **Jangan menamai field
kredensial di luar pola itu**, karena ia akan lolos ke file project yang di-download
dan dibagikan user. `loadProject` juga sengaja mempertahankan `state.api` yang
sedang aktif dan tidak menimpanya dari file.

Key hanya dikirim ke API resmi provider-nya. Jangan tambah endpoint pihak ketiga,
telemetry, atau logging yang menyentuh key.

## `influencer.html` — AI Influencer Studio

Pipeline: `talent → identity → persona → sheet → shoot → render → content → export`.
Bikin virtual influencer dari foto talent: baca fotonya, kunci identitas, susun
persona, bikin character sheet, rencanakan sesi foto, render gambarnya, tulis caption.

**Dua mesin AI terpisah** di `state.api`, dipilih sendiri-sendiri:

- `api.txt` — teks & vision (`gemini`, `claude`, `dinoiki`, `koboillm`, `compat`).
  Dinoiki & KoboiLLM adalah gateway OpenAI-compatible asal Indonesia. Base URL
  bawaan sudah dari dashboard user — Dinoiki `https://ai.dinoiki.com/v1`,
  KoboiLLM `https://lite.koboillm.com/v1` (instalasi LiteLLM) — tapi tetap
  **bisa diedit**, jangan dikunci.

- `api.img` — gambar (`gemini_image`, `chat_image`, `openai_image`, `compat_image`,
  `manual`). Yang menentukan bukan cuma modelnya, tapi **apakah foto acuan bisa
  ikut terkirim** — tanpa itu anchor tidak bekerja dan wajah melenceng:

  | mode | endpoint | foto acuan |
  |---|---|---|
  | `gemini_image` | `models/*:generateContent` | ya, `inlineData` |
  | `chat_image` | `{base}/chat/completions` | ya, `image_url` data URI |
  | `openai_image` | `/images/edits` bila ada acuan **dan** model cocok `/gpt-image/i`, selain itu `/images/generations` | ya, multipart |
  | `compat_image` | `/images/generations` | **tidak** |

  `chat_image` ada karena gateway LiteLLM (KoboiLLM) menyajikan model gambar
  Gemini lewat `/chat/completions`. Bentuk balasannya belum seragam antar-versi,
  jadi `pickChatImage()` sengaja memeriksa empat kemungkinan: `message.images[]`,
  `message.content[]` bertipe `image_url`, data URI di dalam string, dan
  `data[0].b64_json`. Parameter `modalities` juga diturun-tanggakan karena
  sebagian gateway menolaknya.

`fetchModels(base,key)` menarik `GET {base}/models` supaya user tidak perlu
mengetik nama model; hasilnya masuk `MODEL_CACHE` (dikunci per base URL, sengaja
di luar `state` supaya tidak ikut file project) lalu dirender sebagai `<datalist>`.
**Jangan me-render ulang modal API saat kolom Base URL `change`** — node tombol
"Tarik daftar" ikut terganti tepat saat diklik dan klik pertamanya hilang.

`MODEL_HINTS` = rekomendasi model per gateway, disusun dari katalog milik user.
Bukan daftar lengkap dan tidak dipakai untuk validasi — `hintsFor(base)` cuma
menebak gateway dari base URL lalu menampilkan chip. Sumber kebenaran tetap
`GET /models`.

### Aturan domain (jangan dilanggar)

- **`identity.identity_lock` bersifat FIXED.** `finalPrompt()` selalu memprepend-nya
  ke setiap prompt gambar. Tanpa itu tiap foto menghasilkan orang berbeda. AI
  penyusun shot **dilarang** menulis deskripsi wajah di `shots[].prompt` — prompt
  adegan hanya boleh berisi outfit, lokasi, pose, ekspresi, cahaya, komposisi.
- **Mode `locked` (wajah persis) hanya boleh aktif setelah `talent.consent`.**
  `refsForImage()` memeriksa ulang consent sebelum mengirim foto. Jangan longgarkan.
- **Mode `inspired` tidak pernah mengirim foto orang asli ke mesin gambar.**
  Konsistensi datang dari identity lock, lalu dari `state.anchor` — hasil render
  pertama yang otomatis jadi acuan wajah untuk render berikutnya.
- Field kredensial: `geminiKey`, `claudeKey`, `dinoikiKey`, `koboiKey`, `compatKey`,
  `imgGeminiKey`, `imgOaKey`. Semua berakhiran `Key` supaya kena filter ekspor.

Whitelist restore ada di konstanta `KEEP`. Gambar hasil render **tidak** ikut file
project kecuali user mencentang toggle-nya (`withImages`) — ini berlaku untuk
`shots[].imgs`, `sheet.panels[].imgs`, `sheet.grid.cells[].img`, **dan**
`sheet.grid.sheetImg`; kalau menambah tempat penyimpanan gambar baru, tambahkan
juga di `saveProject`.

### Tahap `sheet` — character sheet

Lembar acuan tetap yang dipakai ulang oleh semua tahap sesudahnya. `buildContext()`
menyuntikkan brand kit + panduan niche ke setiap prompt berikutnya, jadi rencana
foto, caption, dan kalender semuanya menurunkan diri dari sini.

- `SHEET_PANELS` = template panel referensi (turnaround, ekspresi, wardrobe, detail,
  mood board). Prompt-nya **dikunci di kode, bukan dikarang AI**, supaya komposisi
  tiap lembar selalu sama dan bisa dibandingkan antar-generate.
- Panel bertanda `noIdentity` (mood board) sengaja **tidak** memakai identity lock —
  isinya benda, bukan orang.
- Panel `turnaround` mengambil alih `state.anchor` begitu jadi: empat sudut wajah
  adalah acuan terbaik yang bisa dihasilkan aplikasi ini. `runPanelAll()` merender
  turnaround duluan supaya panel lain sudah punya anchor.
- Tiap panel punya rasionya sendiri (`def.aspek`) — karena itu `generateImage()`,
  `geminiImage()`, dan `openaiImage()` menerima parameter `aspek` yang menimpa
  `state.shoot.aspek`.

### Sistem realisme (aturan user: hasil tidak boleh terlihat AI slop)

Tiga blok masuk ke **setiap** prompt gambar lewat `composePrompt()`:

1. `CAPTURE` — medium tangkap, dari preset `REALISM` (`phone`, `film35`, `prime85`,
   `docu`, `custom`). Menentukan "rasa" fotonya.
2. `REALISM` — `LIFE_DEFAULT`: pori kulit, warna kulit tak rata, asimetri wajah,
   helai rambut lepas, berat bertumpu satu kaki, kerutan sudut mata, pantulan di
   pupil, kain berkerut, cahaya berarah. Ini sumber "aura"-nya.
3. `AVOID` — `ANTI_SLOP_DEFAULT`: plastic skin, wax figure, dead eyes, perfectly
   symmetrical face, HDR, CGI, uncanny valley, catalog stock photo, dst.

Kolom `shoot.kamera` / `shoot.jiwa` / `shoot.negatif` **kosong = pakai default**
(`techOf()`, `lifeOf()`, `negOf()`). Isi user menimpa default.

**Kata `photorealistic` sengaja tidak dipakai di prompt mana pun** — di banyak
model kata itu justru menarik hasil ke render 3D yang licin. Penggantinya
`"a real, unretouched photograph"` + deskripsi medium yang konkret. Jangan
memasukkannya kembali.

`promptShoot()` juga memaksa tiap prompt adegan memuat mikro-aksi, interaksi
lingkungan, detail lingkungan tak sempurna, dan arah cahaya yang jelas — serta
melarang pose tegak menghadap kamera dan kata "perfect/flawless/stunning".

`promptQA()` menilai **tiga skor terpisah**: `skor_identitas`, `skor_realisme`
(khusus mendeteksi gejala AI slop), `skor_teknis`. Kalau model lupa mengirim
`skor` gabungan, `qaImage()` menghitungnya sendiri dari rata-rata ketiganya.

### Grid character sheet — satu kanvas utuh

`state.sheet.grid` = `{cols, rows, aspek, latar, label, cells[], sheetImg}`.
Ukuran sampai 12×12 (144 pose).

Tidak ada model gambar yang sanggup menaruh 144 pose berwajah konsisten dalam
satu kali generate, jadi **tiap sel dirender terpisah** pada resolusi penuh
(identity lock + anchor yang sama) lalu **dijahit di browser** lewat canvas
(`compositeGrid()`). Hasil akhirnya tetap satu file PNG.

- `POSE_LIB` = 48 pose lintas 8 kategori framing (`FB` badan utuh, `34`, `MD`,
  `CU`, `ST` duduk, `AC` aksi, `SO` format sosial, `HD` detail tangan), sengaja
  diselang-seling supaya sel bertetangga tidak mirip. Dikali `POSE_MODS`
  (6 variasi sudut/ekspresi) → cukup untuk 144 sel unik.
- `gridCellSize()` menghitung mundur ukuran sel dari batas 36 juta piksel.
  Tanpa ini grid 12×12 menembus batas canvas browser.
- `runGridAll()` hanya merender sel yang kosong dan bisa dihentikan
  (`gridAbort`) lalu dilanjutkan — 144 panggilan API tidak boleh hangus.
- Membangun ulang grid **memakai ulang** gambar sel yang posenya sama persis,
  dan membatalkan `sheetImg` karena ukurannya tak lagi cocok.

### Fitur render lanjutan

- **QA konsistensi** (`qaImage`) mengirim hasil render balik ke mesin *vision* dan
  membandingkannya dengan identity lock → skor 0–100 + daftar cacat. Butuh
  `img.data`; gambar yang hanya berupa `remoteUrl` tidak bisa diperiksa.
- **Refine** (`runRefine`) adalah image-to-image: gambar sumber dikirim sebagai
  satu-satunya referensi lewat parameter `explicitRefs`, hasilnya jadi varian baru,
  bukan menimpa. Anchor tidak berubah.
- **Stempel label AI** (`stampedURL`) menggambar ulang lewat canvas saat diunduh.
  Gambar `remoteUrl` mencemari canvas → jatuh ke unduhan biasa; gambar di bawah
  240 px dilewati supaya labelnya tidak menutupi frame.

Ekspor tambahan: character sheet sebagai HTML mandiri (gambar ditanam sebagai data
URI) dan kalender konten sebagai CSV ber-BOM.

## Verifikasi perubahan

Tidak ada test runner. Buka di browser, cek console bersih, lalu smoke test.

- `index.html`: setup → input → generate satu tahap AI → ekspor Remotion → cek
  JSON-nya valid.
- `influencer.html`: isi key → upload foto → generate identitas → susun rencana
  foto → render → ekspor project, lalu buka file .json-nya dan pastikan tidak ada
  API key di dalamnya.

Satu error console yang wajar muncul di kedua file saat offline: stylesheet Google
Fonts gagal dimuat. Font jatuh ke `system-ui`, bukan bug.
