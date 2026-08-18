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

Pipeline: `talent → identity → persona → sheet → arc → shoot → render → feed → content → export`.
Bikin virtual influencer dari foto talent: baca fotonya, kunci identitas, susun
persona, bikin character sheet, rencanakan sesi foto, render gambarnya, tulis caption.

**Dua mesin AI terpisah** di `state.api`, dipilih sendiri-sendiri:

- `api.txt` — teks & vision. Dua jalur native (`gemini`, `claude`) dan lima jalur
  OpenAI-compatible yang semuanya lewat peta `OA`: `dinoiki`, `koboillm`, `grok`,
  `openrouter`, `compat`. Base URL bawaan sudah terisi tapi tetap **bisa diedit**,
  jangan dikunci:

  | id | base bawaan |
  |---|---|
  | `dinoiki` | `https://ai.dinoiki.com/v1` |
  | `koboillm` | `https://lite.koboillm.com/v1` (LiteLLM) |
  | `grok` | `https://api.x.ai/v1` |
  | `openrouter` | `https://openrouter.ai/api/v1` |

  Entri `OA` boleh punya `headers` untuk header tambahan per provider —
  OpenRouter memakainya untuk `X-Title`. Menambah gateway baru cukup satu baris
  di `OA` + satu di `TXT_PROVIDERS` + field `*Base`/`*Key`/`*Model` di `state.api`;
  penarikan daftar model dan chip rekomendasi ikut otomatis.

- `api.img` — gambar (`gemini_image`, `chat_image`, `xai_image`, `openai_image`,
  `compat_image`, `free`, `manual`). Yang menentukan bukan cuma modelnya, tapi
  **apakah foto acuan bisa ikut terkirim** — tanpa itu anchor tidak bekerja dan
  wajah melenceng:

  | mode | endpoint | foto acuan |
  |---|---|---|
  | `gemini_image` | `models/*:generateContent` | ya, `inlineData` |
  | `chat_image` | `{base}/chat/completions` | ya, `image_url` data URI |
  | `xai_image` | `/images/edits` bila ada acuan, selain itu `/images/generations` | ya, data URI di **JSON** |
  | `openai_image` | `/images/edits` bila ada acuan **dan** model cocok `/gpt-image/i`, selain itu `/images/generations` | ya, multipart |
  | `compat_image` | `/images/generations` | **tidak** |
  | `free` | `GET image.pollinations.ai/prompt/…` | **tidak** |

  `chat_image` ada karena gateway LiteLLM (KoboiLLM) menyajikan model gambar
  Gemini lewat `/chat/completions`. Bentuk balasannya belum seragam antar-versi,
  jadi `pickChatImage()` sengaja memeriksa empat kemungkinan: `message.images[]`,
  `message.content[]` bertipe `image_url`, data URI di dalam string, dan
  `data[0].b64_json`. Parameter `modalities` juga diturun-tanggakan karena
  sebagian gateway menolaknya.

  `xai_image` **tidak** boleh dilebur ke `openai_image` meski namanya sama-sama
  `/images/edits`: di xAI endpoint itu menerima JSON biasa (`image:{url,type}` atau
  `images:[…]` untuk maksimal 3 acuan), bukan `multipart/form-data` seperti OpenAI.
  Key-nya sama dengan `api.grokKey` di mesin teks, tapi disimpan terpisah di
  `imgXaiKey` supaya user bisa memakai gateway lain untuk teks.

  `free` (Pollinations) ada supaya user bisa menguji prompt tanpa membakar kuota
  berbayar. Ia **tidak** bisa menerima gambar, jadi mode ini melanggar dasar sistem
  konsistensi wajah — itu bukan bug yang bisa ditambal, dan UI **wajib**
  mengatakannya. `imgSendsRefs()` adalah sumber kebenarannya; `noRefWarn()` memasang
  banner di tahap Character Sheet dan `viewRender()` mengganti kalimat "anchor aktif"
  supaya app tidak menjanjikan hal yang tidak dikerjakannya. Kalau menambah mesin
  gambar tanpa dukungan acuan, daftarkan juga di `imgSendsRefs()`.

  Prompt lengkap masuk ke path URL, jadi `freeImage()` menolak prompt yang membuat
  URL lewat ~7500 karakter dengan pesan yang bisa dibaca user — kalau tidak, yang
  muncul adalah HTTP 414 yang tak berarti apa-apa.

  **Groq tidak bisa dipakai sebagai mesin gambar** — layanannya inferensi LLM dan
  vision *input*, tanpa endpoint pembuatan gambar. Key Groq tetap berguna di mesin
  teks lewat `compat` (`https://api.groq.com/openai/v1`). Cloudflare Workers AI
  gratis tapi REST API-nya tidak mengirim header CORS, jadi tidak bisa dipanggil
  langsung dari app `file://` ini tanpa Worker proxy sendiri.

  Mesin gambar berbentuk gateway (punya `GET /models`) didaftarkan di
  `imgGateway()` — satu tempat yang dipakai bersama oleh tombol "Tarik daftar" dan
  chip rekomendasi. Mode tanpa gateway mengembalikan `null` dan chip-nya dikosongkan.

`fetchModels(base,key)` menarik `GET {base}/models` supaya user tidak perlu
mengetik nama model; hasilnya masuk `MODEL_CACHE` (dikunci per base URL, sengaja
di luar `state` supaya tidak ikut file project) lalu dirender sebagai `<datalist>`.
**Jangan me-render ulang modal API saat kolom Base URL `change`** — node tombol
"Tarik daftar" ikut terganti tepat saat diklik dan klik pertamanya hilang.

`MODEL_HINTS` = rekomendasi model per gateway, disusun dari katalog milik user.
Bukan daftar lengkap dan tidak dipakai untuk validasi — `hintsFor(base)` cuma
menebak gateway dari base URL lalu menampilkan chip. Sumber kebenaran tetap
`GET /models`.

Chip itu hidup di wadah terpisah (`#rec-txt` / `#rec-img`) dan disegarkan lewat
`refreshRecs()` pada event **`input`** kolom Base URL. Jangan pindahkan ke
`change`: `change` baru menyala saat fokus berpindah — yaitu persis ketika user
mengklik tombol "Tarik daftar" — sehingga node tombolnya ikut terganti dan klik
pertamanya hilang.

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
- Field kredensial: `geminiKey`, `claudeKey`, `dinoikiKey`, `koboiKey`, `grokKey`,
  `orKey`, `compatKey`, `imgGeminiKey`, `imgOaKey`, `imgXaiKey`. Semua berakhiran
  `Key` supaya kena filter ekspor.

Whitelist restore ada di konstanta `KEEP`. Gambar hasil render ikut file project
selama `state.withImages` menyala — **default-nya sengaja menyala**, karena gambar
adalah artefak termahal di app ini (satu render = satu panggilan API berbayar) dan
kehilangannya diam-diam jauh lebih buruk daripada file yang besar. Tombol 💾 Simpan
ada di header dan bisa diklik dari tahap mana pun, jadi user bisa sama sekali tidak
pernah melihat toggle di tahap Ekspor; karena itu `saveProject()` menahan dengan
`confirm()` kalau toggle mati **dan** ada gambar yang akan hilang. Jangan hapus
gerbang itu. Saat toggle mati, yang dibuang adalah — ini berlaku untuk
`shots[].imgs`, `sheet.panels[].imgs`, `sheet.grid.cells[].img`, **dan**
`sheet.grid.sheetImg` — kalau menambah tempat penyimpanan gambar baru, tambahkan
juga di `projectJSON()` dan di `imageCount()`.

Autosave IndexedDB bersifat **lokal per browser dan per mesin, tidak sinkron**.
Satu-satunya cara memindahkan proyek antar komputer adalah file project (.json)
atau arsip ZIP — tahap Ekspor memuat checklist langkahnya.

### Bar progres

`setBusy(b, text, done, total)` menggerakkan bar global di antara header dan body,
jadi ia terlihat dari tahap mana pun tanpa perlu diduplikasi di tiap view.

**`done`/`total` hanya boleh diisi operasi yang benar-benar tahu jumlah langkahnya** —
render semua foto, grid, panel, audit, QA massal, pasca-proses massal, pengemasan ZIP.
Untuk panggilan tunggal, kemajuan di sisi server tidak bisa diketahui, jadi bar-nya
bergerak **tanpa persentase** dan yang ditampilkan adalah detik berjalan. Menampilkan
persen karangan di situ hanya memindahkan tebakan dari user ke aplikasi — persis yang
fitur ini ada untuk menghilangkan. Jangan pernah mengarang angka di jalur itu.

`progDone`/`progTotal` bertahan selama batch berlangsung. Pesan tanpa hitungan yang
muncul **di tengah** batch (retry jaringan di `postJSON`, render ulang gerbang QA di
`renderShot`) hanya mengganti labelnya — kalau ia mengosongkan persentase, bar berkedip
balik ke nol persis saat user paling butuh kepastian. Keduanya dinolkan di
`setBusy(false)` supaya tidak bocor ke operasi tunggal berikutnya.

Teks header sengaja diringkas jadi `Memproses n/total`: detailnya sudah dibawa bar, dan
mengulang kalimat panjang di header membuat barisnya melipat.

### Kepatuhan & pelabelan

Ada di kartu tahap Ekspor, dan dibelah dua **dengan sengaja**:

- **Temuan yang dihitung** (`patuhAudit()`) — berasal dari isi proyek sendiri, jadi app
  boleh memastikannya: penanda AI mati, caption yang tidak memuat kata penanda, cap
  label gambar mati, mode `locked` tanpa catatan izin, foto di bawah ambang QA.
  Pemeriksaan caption membaca **teksnya**, bukan niatnya, dan UI mengatakan itu.
- **Daftar centang platform** (`PATUH`) — app **tidak boleh** mengklaim ini akurat.
  Aturan pelabelan media sintetis berubah cepat dan berbeda tiap platform, dan aplikasi
  ini tidak memantaunya. Banner di atas kartu menyatakan itu terang-terangan: catatan
  kerja, bukan nasihat hukum, verifikasi ke kebijakan resmi. **Jangan pernah** mengubah
  daftar itu menjadi klaim bahwa aturannya masih berlaku hari ini.

`state.patuh.cek` menyimpan centang user supaya keputusannya ikut di file project.
Daftar disaring oleh `state.sheet.platforms`; kategori `umum` selalu ikut.

`provenanceText()` dan `consentText()` menghasilkan catatan kerja, bukan bukti.
Keduanya menyebutkan batas dirinya sendiri di dalam dokumennya — catatan produksi bukan
tanda tangan kriptografis, catatan izin bukan dokumen hukum. `provenanceText()` menyebut
mesin dan model yang dipakai tapi **tidak pernah** menyentuh API key.

Keduanya ikut di ZIP; `addText()` melewati string kosong, jadi catatan izin tidak muncul
di proyek mode `inspired`.

### Tahap `arc` — alur hidup, mengunci waktu

Wajah dikunci lewat identity lock, dunia lewat kitab, komposisi lewat simulasi feed.
Yang tersisa adalah **waktu**. Akun yang isinya sama persis selama berbulan-bulan —
outfit itu-itu saja, tidak pernah ke mana-mana, tidak pernah terjadi apa pun — sama
mencurigakannya dengan wajah yang berubah-ubah.

`state.arc` = `{bulan, mulai, tema, babak:[]}`; babak =
`{id, nama, minggu, cerita, perubahan, konten, musim}`.

Field **`perubahan`** adalah porosnya: apa yang berubah **dan terlihat di foto** pada
babak itu (potongan rambut, satu barang baru, warna dominan, cuaca). Tanpa itu alurnya
cuma catatan yang tidak pernah terlihat siapa pun. `promptArc()` mewajibkannya, sekaligus
**melarang** mengubah wajah, bentuk tubuh, dan tanda permanen — perubahan waktu tidak
boleh menabrak kunci identitas.

Prompt-nya juga melarang drama besar (patah hati sinetron, mendadak kaya) dan
memerintahkan memanfaatkan konteks Indonesia — Ramadan, Idulfitri, 17 Agustus, musim
hujan — kalau `mulai` menyentuh rentangnya. Mengabaikan musim membuat akun terasa tidak
hidup di tempat yang sama dengan audiensnya.

`arcBrief()` menyodorkan seluruh babak ke `promptShoot()` **dan** `promptKalender()`,
jadi sesi dan postingan menurunkan diri dari alur alih-alih mengambang di luar waktu.

`sesi.babak` menyimpan id babak asalnya. Tombol "buat sesi dari babak ini" menyalin
cerita/perubahan/musim ke `sesi.catatan` — itu yang membuat alur bisa ditindaklanjuti,
bukan cuma dibaca. Menghapus babak **tidak** menghapus sesinya; sesinya dilepas
(`babak=''`), dan `normalizeState()` membersihkan rujukan yatim.

Menambah tahap berarti **menomori ulang** `head('Tahap NN', …)` di semua view sesudahnya.
Lakukan menurun (10→9→8…) supaya penggantian tidak saling tabrak, dan ingat sebagian view
punya dua `head()` (kondisi kosong + normal). smoke19 memeriksa nomor di judul tiap view
cocok dengan `STEPS[].no`, jadi kelalaian di sini ketahuan.

### Roster — banyak influencer di satu browser

IndexedDB tidak lagi memakai satu slot. Sekarang: `proj:<projId>` per influencer +
satu record `roster` berisi **metadata saja** (`{id, nama, t, nFoto, nImg, nSesi,
thumb}`). Metadata dipisah dengan sengaja — menampilkan daftar tidak boleh memuat
ratusan MB gambar milik proyek yang tidak sedang dibuka. `thumb` digambar ulang lewat
canvas ke 96 px JPEG, jadi beberapa kB, bukan beberapa MB.

**`state.projId` sengaja TIDAK ada di `KEEP`.** Memuat file project dari mesin lain
harus jadi entri roster baru, bukan menimpa proyek yang kebetulan ber-id sama.
`projNama` ikut di `KEEP` supaya labelnya tetap terbawa bersama file. Invarian ini
diuji — kalau `projId` masuk `KEEP`, smoke13 gagal.

`flushProyek()` menyimpan proyek yang sedang dibuka **sebelum** berpindah; tanpa itu
kerja beberapa detik terakhir hilang tiap kali user ganti influencer. `projectEpoch`
dinaikkan di setiap perpindahan, bukan cuma di reset — autosave yang sedang jalan
memegang snapshot proyek lama dan tulisannya harus dibuang kalau mendarat belakangan.

`migrasiSlotLama()` memindahkan slot `autosave` versi satu-proyek ke roster sekali saat
boot lalu menghapusnya. Tanpa ini pekerjaan user yang sudah ada tampak lenyap begitu
app diperbarui.

Dua perubahan makna yang sengaja:

- **Reset** mengosongkan isi proyek tapi **mempertahankan entri roster-nya** — yang
  user minta adalah mengosongkan influencer ini, bukan menghapusnya dari daftar.
  Menghapus dari daftar punya tombolnya sendiri di 👥 Proyek.
- **"Buang"** di banner pemulihan hanya menutup tawaran dan mengosongkan `roster.aktif`.
  Ia **tidak lagi menghapus slot**: dengan adanya roster, membuang influencer diam-diam
  lewat tombol itu akan jadi kehilangan yang tak terduga.

`hapusProyek()` yang menyasar proyek yang sedang dibuka juga mengosongkan memori —
kalau tidak, autosave berikutnya menulis ulang slot yang baru saja dihapus.

### Reset — isi proyek vs setup

`state` dibelah dua secara sengaja:

- **Isi proyek** dibuat oleh `freshProject()` — `talent`, `uploads`, `identity`,
  `persona`, `shoot`, `sheet`, `kalender`, `shots`, `anchor`, `dirty`. Semua ini
  milik satu influencer.
- **Setup** ada di objek `state` itu sendiri — `api`, `post`, `qaGate`, `stampAI`,
  `withImages`, `usage`. Milik mesin/preferensi user, bukan milik influencer-nya.

`state` dibangun sebagai `Object.assign(freshProject(), { …setup… })`. **Field isi
proyek yang baru wajib ditaruh di `freshProject()`, bukan di literal setup** —
kalau tidak, ia lolos dari reset dan bocor ke proyek berikutnya. Invariannya
diuji: setiap key `freshProject()` harus ada di `KEEP`.

`resetProject()` menimpa **per key**, bukan mengganti objek `state`, supaya
referensi yang sudah dipegang closure lain tetap menunjuk ke state yang sama.
`usage.harga` dipertahankan (properti gateway, bukan proyek) sementara
penghitungnya dinolkan.

Mengganti foto talent bukan cuma mengganti satu tahap: identity lock, persona,
character sheet, rencana foto, dan semua render adalah turunan dari talent lama.
**`state.anchor` adalah kebocoran paling berbahaya** — ia dikirim ke mesin gambar
di setiap render tanpa pernah muncul di form, jadi wajah lama bisa ikut ke proyek
baru tanpa user melihat apa pun.

Dua hal yang jangan dilonggarkan:

- **Slot autosave WAJIB ikut dihapus.** Tanpa itu, membuka app lagi menawarkan
  memulihkan proyek yang baru dibuang — kebocoran lewat pintu belakang, persis
  yang tombol ini seharusnya mencegah.
- **`projectEpoch`.** Autosave yang sudah berjalan sebelum reset memegang snapshot
  proyek lama; kalau tulisannya mendarat setelah slot dihapus, snapshot lama hidup
  kembali. `resetProject()` menunggu `autoRunning` selesai, dan `autosaveNow()`
  memeriksa `projectEpoch` sesudah put sebagai jaring kedua karena transaksi yang
  sudah dikirim tidak bisa dibatalkan.

Modal konfirmasinya merinci **dua kolom** — apa yang dihapus (dengan jumlah dan
perkiraan biaya API untuk membuatnya ulang) dan apa yang dipertahankan. Itu bukan
hiasan: tanpa kolom kedua, user akan menghindari tombolnya karena takut key-nya
hilang. Tombol "💾 Simpan dulu" sengaja **tidak** menutup modal. Semua jalan keluar
yang tidak sengaja — klik latar, Escape — berarti **batal**, bukan lanjut.

### Sesi pemotretan — arsitektur prompt bertingkat

Prompt gambar punya **tiga tingkat, semuanya dikunci di kode, bukan dikarang AI**:

| tingkat | blok | ruang lingkup | sumber |
|---|---|---|---|
| 1 | `SUBJECT` | seumur proyek | `identity.identity_lock` |
| 2 | `SETTING` | satu sesi | `settingLine(sesi)` |
| 3 | `SCENE` | satu foto | `shots[].prompt` |

Alasan tingkat 2 ada sama persis dengan alasan tingkat 1 ada: kalau outfit ditulis
ulang oleh AI di tiap foto, ia berubah sedikit demi sedikit — kemeja linen krem jadi
blus katun putih — dan feed-nya langsung terbaca palsu. Yang membuat feed AI ketahuan
**bukan cacat di masing-masing gambar, melainkan hubungan antar gambar**: orang
sungguhan tidak ganti baju dan pindah kota di setiap postingan.

Karena itu `promptShoot()` **melarang** AI menyebut outfit, pakaian, lokasi, rambut,
waktu, atau arah cahaya di `shots[].prompt` — sama seperti larangan menulis wajah.
`shots[].prompt` hanya boleh berisi pose, mikro-aksi, ekspresi, framing, dan satu
detail lingkungan. Jangan longgarkan; ini poros seluruh fitur.

`state.sesi[]` = `{id, nama, outfit, lokasi, waktu, cahaya, rambut, catatan}`.
`shots[].sesi` menyimpan id-nya; `''` berarti **foto lepas** — tetap bisa dirender,
hanya tidak mendapat blok `SETTING`.

**Peran foto (`PERAN`).** Feed asli bukan kumpulan foto juara semua — justru itu yang
membuatnya terbaca sebagai katalog. Empat peran: `hero`, `filler`, `objek`, `cermin`,
dengan porsi di `peranMix()`.

Peran `objek` (frame tanpa orang) punya `noIdentity` dan diperlakukan berbeda di
**empat** tempat. Keempatnya wajib — masing-masing pernah jadi sumber bug yang sama
di grid character sheet (`nf`):

1. `finalPrompt()` tidak memasang identity lock **maupun** blok REALISM (keduanya
   bicara soal kulit dan wajah), dan memakai `OBJEK_TAIL`.
2. `renderShot()` mengirim `explicitRefs=[]` — anchor adalah wajah; mengirimkannya
   ke frame benda membuat model memaksa wajah masuk frame.
3. `renderShot()` melewati gerbang QA — `qaImage()` membandingkan hasil dengan
   identity lock, jadi foto meja selalu dinilai nol dan memicu render ulang berbayar.
4. Hasilnya **tidak boleh** jadi `state.anchor`. Anchor harus selalu wajah.

Frame `objek` juga yang paling murah dan paling meyakinkan sekaligus — itu bukan
efek samping, itu alasan porsinya besar.

**Kompatibilitas mundur** ditangani di dua tempat, keduanya sengaja memaafkan:

- `normalizeState()` — project lama menyimpan `outfit`/`lokasi`/`pencahayaan` di
  level foto. Nilainya **tidak dibuang**, ditempelkan ke `prompt` supaya foto lama
  tetap merender adegan yang sama; fotonya jadi foto lepas.
- `adoptShoot()` — balasan AI bentuk lama (`{"shots":[…]}` datar) dibungkus jadi satu
  sesi supaya tidak ada foto yatim.

Menghapus sesi **tidak** menghapus fotonya — foto berisi gambar hasil render yang
mahal. Foto-fotonya dilepas jadi foto lepas; user yang memutuskan nasibnya.

### Kit acuan wajah — satu acuan per sudut

Sistem anchor lama memakai **satu** gambar sebagai acuan untuk semua render. Kalau
anchor itu frontal — dan hampir selalu begitu, karena render pertama biasanya frontal —
maka setiap foto yang minta profil, tiga-perempat, atau dagu terangkat memaksa model
**mengarang wajah dari sudut yang belum pernah ia lihat**. Di situlah morph dan wajah
melenceng lahir. Bukan di daftar posenya.

`ACUAN` = 8 frame identitas (bukan pose konten): depan rambut diikat, 3/4 kiri & kanan,
profil kiri & kanan, mendongak, menunduk, makro tekstur. `state.acuan.frames[]` menyimpan
hasilnya.

`anchorFor(sudut)` memilih acuan yang sudutnya paling dekat lewat `SUDUT_DEKAT`. Urutan
kedekatannya bukan sembarang: profil ditolong tiga-perempat lebih dulu, **bukan** frontal,
karena frontal tidak memuat garis samping wajah sama sekali. Kalau kit belum ada, ia jatuh
ke `state.anchor` — fitur ini **menambah**, tidak menggantikan jalur lama.

`sudutOf(sh)` memakai field `sudut` dari planner kalau ada, kalau tidak menebak dari kata
kunci. Di `SUDUT_KATA`, pola **wajah menoleh ke kamera diperiksa lebih dulu** daripada
'belakang': *"head turned back to camera"* dan *"glancing back over the shoulder"* berarti
wajahnya terlihat. Kalau 'belakang' diperiksa duluan, frasa itu salah terbaca dan fotonya
dapat acuan yang keliru. Alias `34b` sengaja paling belakang supaya kata *three-quarter*
(yang biasanya menyebut BADAN) tidak mengalahkan petunjuk sudut wajah yang lebih spesifik.

**Rambut diikat** di frame depan dan kedua profil. Garis rambut, telinga, dan leher tidak
pernah terlihat di foto konten biasa — dan apa yang tidak pernah terlihat akan dikarang
ulang setiap kali render membutuhkannya.

`runAcuanAll()` merender frame **depan lebih dulu**, lalu memakainya sebagai acuan untuk
tujuh frame lainnya supaya kitnya sendiri konsisten. Frame depan juga menjadi
`state.anchor`.

`poseUrut()` menyusun ulang urutan `POSE_LIB` untuk `buildGridCells()`. Dulu grid mengambil
`POSE_LIB[0..n]` berurutan, dan dua belas entri pertama kebetulan hampir semuanya
menghadap depan — jadi grid 4×3 tidak pernah memuat profil, sudut atas/bawah, maupun
punggung. Sekarang sudut diselang-seling sejak sel pertama; grid besar tetap kebagian
seluruh pustaka.

### Kitab kesinambungan — mengunci dunia, bukan cuma wajah

`state.kitab` = `{outfit:[], lokasi:[], props:[], tanda:[]}`, tiap entri
`{id, nama, teks, catatan, img, kirim}`. `teks` **wajib bahasa Inggris** — ia masuk
mentah ke prompt gambar; `nama` cuma label untuk daftar di UI.

Empat kategori, **tiga titik suntik berbeda**, dan itu bukan detail sepele:

| kategori | masuk ke | ruang lingkup |
|---|---|---|
| `tanda` | `lockLine()` → blok **SUBJECT** | menempel pada orangnya, ikut di setiap foto berisi dia |
| `outfit`, `lokasi`, `props` | `settingLine()` → blok **SETTING** | milik satu sesi |

Tanda permanen sengaja **bukan** SETTING: tato tidak berganti waktu sesi berganti.
Menaruhnya di SETTING berarti ia hilang di foto lepas dan di sesi yang lupa memilihnya.

**Rujukan, bukan salinan.** `sesi.outfitRef` / `sesi.lokasiRef` / `sesi.props[]` menyimpan
id entri. `pakaiRef()` menyelesaikannya ke `teks`, dengan teks bebas milik sesi sebagai
cadangan. Itu inti fiturnya: mengedit satu entri lemari pakaian ikut memperbaiki semua
sesi yang memakainya. Kalau ini diubah jadi salinan, fiturnya kehilangan alasan ada.

`normalizeState()` membuang rujukan ke entri yang sudah terhapus — kalau tidak, sesi
diam-diam terkunci ke teks kosong. Teks bebas sesi tetap jadi jaring pengamannya, dan
tombol Hapus memperingatkan berapa sesi yang terdampak sebelum menghapus.

`kitabBrief()` menyodorkan isi kitab ke `promptShoot()` supaya planner **memakai ulang**
barang yang sudah ada, bukan mengarang lemari pakaian baru tiap kali — orang sungguhan
punya sedikit baju yang dipakai berulang. AI merujuk lewat **nama** (id tidak pernah ia
lihat); `adoptShoot()` mencocokkannya jadi id, dan kalau tidak ketemu jatuh ke teks
bebas — jangan diam-diam mengunci ke entri yang salah.

`refsForShot(sh)` menggantikan `refsForImage()` di `renderShot()`: anchor wajah dulu,
lalu foto acuan entri kitab yang aktif di sesi itu **dan** ditandai `kirim`, dipotong 4.
Frame `objek` tidak menerima wajah sama sekali tapi **tetap** menerima acuan benda dan
tempatnya — justru itu isinya.

Gambar entri kitab ikut `imageCount()`, `imageWeight()`, penyaringan `projectJSON()`
saat `withImages` mati, dan folder `04-kitab/` di ZIP. Batasnya 4 MB per entri (lebih
kecil daripada foto talent) karena puluhan entri berfoto besar akan membengkakkan file
project yang harus dibawa antar-komputer.

**Foto acuan entri dibuat aplikasi, bukan di-upload user.** Foto sembarangan justru
merusak render: ia jadi acuan visual yang bertengkar dengan teksnya. `kitabImgPrompt()`
menggambar dari `teks` entri itu sendiri, jadi gambarnya tidak mungkin bertentangan
dengan kalimat yang melahirkannya. `KITAB_TAIL` memberi tiap kategori bingkai berbeda —
outfit sebagai flat-lay, lokasi sebagai ruangan kosong, props sebagai benda terpakai,
tanda sebagai makro kulit — dan **keempatnya menegaskan tanpa orang di frame**.

Karena itu semua frame acuan memakai `noIdentity` + `noLife` dan `explicitRefs=[]`:
tidak ada identity lock, tidak ada blok kulit, dan **anchor wajah tidak pernah dikirim**.
Rasionya dipaksa `1:1`.

`renderKitabImg()` menyalakan `kirim` otomatis **kecuali** untuk `tanda`: makro kulit
bukan elemen adegan, dan menyelipkannya ke slot acuan hanya menggeser jatah anchor wajah
di `refsForShot()` yang dibatasi 4. Teks tanda sudah masuk blok SUBJECT lewat
`tandaLine()`.

`runKitab()` menawarkan langsung membuat fotonya setelah entri tersusun — dipisah dari
blok `finally` supaya `setBusy` sudah lepas dan `confirm()`-nya tidak muncul di balik
layar sibuk. Tombol `+` (upload manual) tetap ada untuk user yang punya foto asli.

`runKitab()` **menambah, tidak menimpa** — entri yang sudah dipakai sesi tidak boleh
hilang hanya karena tombolnya ditekan dua kali; duplikat disaring lewat `teks` yang sama.

### Audit konsistensi menyeluruh

`promptQA()` membandingkan foto dengan **teks** identity lock. Itu berguna, tapi teks
tidak pernah bisa menangkap sebuah wajah — dua orang berbeda bisa sama-sama cocok
dengan "perempuan Asia Tenggara, wajah oval". Yang benar-benar mengungkap drift adalah
membandingkan foto dengan **foto**.

`promptQAPair()` + `qaPair(img)` mengirim **anchor dan kandidat berdampingan** dalam
satu panggilan. Urutannya penting: anchor selalu dikirim lebih dulu supaya cocok dengan
"FOTO 1 / FOTO 2" di prompt. Biayanya sama — satu panggilan per foto — tapi jauh lebih
sensitif. Hasilnya ditandai `qa.mode==='banding'` dan menambah field `qa.beda[]`.

Prompt-nya **menyuruh mengabaikan** pose, ekspresi, sudut, pakaian, rambut, riasan, dan
cahaya, lalu menyebut secara eksplisit ciri yang tidak boleh berbeda pada orang yang
sama (jarak antar-mata, bentuk hidung, garis rahang, letak tahi lalat). Tanpa daftar itu
model menilai "mirip secara umum" dan hampir selalu meluluskan.

`auditTargets()` melewati frame `objek` — di dalamnya tidak ada wajah untuk dibandingkan,
dan memeriksanya hanya membakar panggilan berbayar untuk skor yang selalu nol. Anchor
sendiri juga dikeluarkan; membandingkannya dengan dirinya sendiri tidak berarti apa-apa.

`auditAll(ulang)` bisa dihentikan (`auditAbort`) dan **melewati foto yang sudah punya
skor** kecuali diminta ulang — 30 foto berarti 30 panggilan, jadi jangan pernah
menjalankannya diam-diam. `confirm()` menyebut jumlah panggilan dan perkiraan biayanya
lebih dulu.

**Diagnosis tingkat kumpulan** adalah bagian yang tidak dimiliki pemeriksaan per foto.
`auditRingkas()` menghitung rata-rata, sebaran, jumlah di bawah ambang, dan **pola cacat
berulang** lewat `CIRI` (penghitungan kata kunci per bagian wajah). Tiap ciri dihitung
**sekali per foto**, bukan sekali per kalimat — kalau tidak, satu foto yang cerewet
mendominasi seluruh diagnosis.

Aturan yang membuat fitur ini berguna: kalau satu ciri melenceng di **≥40% foto**, itu
bukan kesalahan render satu per satu melainkan **identity lock yang kurang spesifik**,
dan UI mengatakannya — arahkan user memperbaiki Tahap 02, bukan menambal foto satu-satu.
Sebaran skor >18 diperingatkan terpisah karena penyebabnya berbeda: biasanya anchor
berganti di tengah jalan.

Ambangnya memakai `state.qaGate.min` yang sudah ada — sengaja **satu konsep ambang**,
bukan dua yang bisa berbeda diam-diam.

### Tahap `feed` — simulasi feed

Feed dibaca sebagai satu kesatuan, bukan foto per foto. Dua cacat hanya muncul di
tampilan grid dan tidak pernah terlihat saat memeriksa gambar satu-satu: foto satu
sesi yang menggerombol berurutan (terbaca sebagai satu kali unggah massal) dan
kecerahan antar-ubin yang melompat-lompat.

`feedAudit()` menghitung empat temuan **dari gambarnya sendiri di browser** — sesi
menggerombol, porsi frame tanpa orang, sebaran kecerahan, dan ubin bertetangga yang
warnanya nyaris kembar. Tidak ada panggilan API: gratis, dan hasilnya sama tiap kali.
Tiap temuan menyebut posisinya supaya bisa langsung ditindak. `autoUrut()` menyelang-
nyeling sesi dengan mengambil dari grup terbanyak yang bukan grup sebelumnya.

Dua hal yang jangan dilonggarkan:

- **`state.feed.urutan` menyimpan id foto, bukan indeks.** Indeks bergeser begitu satu
  foto dihapus dan urutannya jadi salah diam-diam. Karena itu `shots[]` punya `id`
  (`shotId()`), diisi otomatis di `adoptShoot()` dan ditambal di `normalizeState()`
  untuk project lama. `feedOrder()` menambal sendiri: id yang hilang dibuang, foto
  baru menyusul di belakang.
- **Rata-rata warna disimpan di `AVG` (WeakMap), di luar `state`.** Ia turunan dari
  gambar — tidak boleh ikut file project maupun autosave. `hitungAvg()` berjalan di
  latar lalu memanggil `render()` sekali saat selesai; gambar `remoteUrl` mencemari
  canvas jadi dilewati diam-diam.

Menggeser ubin menyimpan juga urutan foto yang **belum** dirender, kalau tidak foto
itu melompat ke belakang begitu selesai dirender.

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

### Disiplin prompt adegan — dipelajari dari prompt UGC kelas produksi

Sebuah contoh iklan UGC AI (arcads.ai, model video, 3 gambar acuan + satu prompt panjang)
memperlihatkan disiplin yang jauh lebih ketat daripada yang dulu dihasilkan `promptShoot()`.
Yang **bisa dipindahkan ke pipeline gambar diam** sudah diserap:

- **Geometri kamera, bukan cuma jarak.** Prompt harus menyatakan di mana kamera berada
  terhadap subjek (sisi + derajat + apakah di depan atau menyerong), lalu arah pandang
  terhadap lensa. `"close-up"` saja tidak memberi tahu model di mana kamera berdiri —
  itu sebabnya hasil lama terlihat seperti pas foto.
- **Isi frame-kiri dan frame-kanan.** Ruang yang tidak disebut akan diisi model dengan
  latar generik.
- **Satu detail akibat per foto** — bukan cacat acak, melainkan bekas dari sesuatu yang
  baru saja terjadi di adegan itu (bekas kacamata di batang hidung, helai yang baru
  tertiup). Ini yang membuat foto punya sebelum-dan-sesudah, bukan pose dari kekosongan.
- **Dilarang bergaya iklan.** Produk tidak diangkat ke kamera, tidak dipamerkan labelnya,
  tidak sejajar wajah — dipegang rendah dan dekat badan. Pose menyodorkan produk adalah
  penanda iklan yang paling cepat terbaca.
- **Kamera adalah ponselnya.** Foto bergaya selfie tidak boleh menampilkan ponsel; lengan
  pemegangnya memendek tajam di tepi frame. Orang tidak bisa terlihat memegang ponsel
  sementara ponsel itu yang memotretnya.
- **Sebut lampu yang MATI.** Menyatakan sumber yang tidak menyala mencegah model
  menambahkan cahaya kedua yang mengacaukan arah bayangan.

Yang **tidak** bisa dipindahkan: koreografi berstempel waktu sub-detik dan aturan "tidak
ada dead air" — keduanya milik ranah video. Jangan menyalinnya ke prompt gambar diam.

### Blok fisika adegan — target deteksi 2026

Riset deteksi mutakhir mengubah prioritas: tanda-tanda lama (jari enam, teks kacau,
kulit lilin) sebagian besar **sudah diperbaiki model**. Yang masih konsisten gagal dan
jadi sandaran pemeriksa sekarang adalah **fisika adegan** — arah cahaya, kekonsistenan
bayangan, pantulan, geometri latar — plus komposisi yang terlalu rapi.

`PHYSICS_DEFAULT` masuk ke **setiap** prompt gambar lewat `composePrompt()` dan berlaku
untuk frame berisi orang **maupun** frame benda: bayangan dan pantulan yang salah sama
menyoloknya di foto meja seperti di potret. Hanya `noPhysics` yang mematikannya.

`ANTI_SLOP_DEFAULT` diperluas dengan cacat fisika (bayangan arah kacau, bayangan kontak
hilang, pantulan tidak cocok, garis latar bengkok, komposisi tengah sempurna) — daftar
lama tetap dipertahankan karena masih relevan untuk model yang lebih lemah.

`promptQA()` dan `promptQAPair()` ikut menilai fisika, bukan cuma kulit. `promptShoot()`
mewajibkan field cahaya sesi punya **geometri** (satu sumber + arahnya), bukan kata sifat
seperti "golden hour" — kata sifat tidak memberi model geometri untuk digambar.

### Anchor: hanya dari frame satu subjek

Beberapa tempat dulu berebut `state.anchor`, dan dua di antaranya menyetelnya ke **lembar
gabungan** (panel turnaround, panel lain) — empat sampai enam wajah dalam satu frame.
Mengirimkan itu sebagai acuan membuat model harus menebak wajah yang mana, dan itu ikut
memperburuk drift. Sekarang anchor hanya boleh datang dari frame **satu subjek**:

- `renderAcuan('depan')` — jalur resmi
- `renderShot()` — hasil render foto, kecuali peran `objek`
- `renderGridCell()` — kecuali sel ber-flag `nf` (detail tangan/sepatu tidak punya wajah)
- tombol ⚓ manual

Panel `detail` sebagian besar digantikan Kit acuan wajah; ia dipertahankan karena tetap
enak dibaca manusia sebagai satu lembar, tapi labelnya menunjuk ke Kit acuan supaya user
tidak mengandalkannya untuk konsistensi.

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

**Jangan pernah menulis "seamless studio backdrop", "flat even lighting", atau
"no cast shadows" di prompt orang.** Dua itu pemicu terkuat tampilan plastik —
lebih kuat daripada blok anti-slop yang melawannya, karena lebih spesifik. Baik
`SHEET_PANELS` (lewat `REF_BG`) maupun `GRID_BG.studio` memakai dinding cat nyata
bertekstur dengan cahaya jendela berarah; kata `seamless` hanya boleh muncul di
daftar AVOID. Pernah ada regresi persis di sini: template panel meminta studio
sementara blok realisme meminta sebaliknya, dan template yang menang.

`LIFE_DEFAULT` menyebut cacat **konkret dan berlokasi** (pori di hidung/pipi/dahi,
kemerahan di sekitar cuping hidung, bekas jerawat samar, satu mata sedikit lebih
kecil, gigi agak kekuningan dan tidak rata). Deskripsi umum seperti "realistic
skin" tidak bekerja — model butuh benda spesifik untuk digambar.

`promptIdentity()` melarang AI memakai kata *beautiful / gorgeous / stunning /
flawless / perfect / model / influencer* di `identity_lock`, dan mewajibkan minimal
dua ciri tak sempurna permanen. Kosakata glamor menarik hasil ke wajah katalog.

### Pasca-proses realisme (canvas)

`realismPass(img, state.post)` adalah satu-satunya bagian yang hasilnya **tidak
bergantung model**. Menambahkan empat jejak yang selalu ada di foto kamera dan
tidak pernah ada di keluaran model: noise sensor berbobot luminansi (bayangan
lebih berisik), aberasi kromatik yang menguat ke tepi frame, vignette radial, dan
**re-encode ke JPEG** — yang terakhir paling menentukan, karena PNG bersih
sempurna adalah tanda paling gampang terbaca sebagai hasil AI.

Hasilnya jadi **varian baru**, gambar asli tidak ditimpa. Karena mimeType berubah
jadi `image/jpeg`, semua penamaan file lewat `imgExt(img)` — jangan hardcode
`.png` lagi di unduhan atau di `exportZip`.

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
- Pose `HD` (detail tangan/kaki) diberi flag `nf` dan **tidak** membawa identity
  lock. Blok itu mendeskripsikan wajah, dan model akan memaksa wajah masuk frame
  sehingga sel "Detail tangan" keluar sebagai potret. Identitas untuk frame itu
  dijaga lewat warna kulit, bentuk kuku, dan aksesori — bukan wajah.
- Tombol ⚓ per sel menyetel `state.anchor` tanpa merender ulang, supaya user bisa
  memilih sel yang wajahnya paling pas sebagai acuan render berikutnya.
- `gridCellSize()` menghitung mundur ukuran sel dari batas 36 juta piksel.
  Tanpa ini grid 12×12 menembus batas canvas browser.
- `runGridAll()` hanya merender sel yang kosong dan bisa dihentikan
  (`gridAbort`) lalu dilanjutkan — 144 panggilan API tidak boleh hangus.
- Membangun ulang grid **memakai ulang** gambar sel yang posenya sama persis,
  dan membatalkan `sheetImg` karena ukurannya tak lagi cocok.

### Autosave (IndexedDB)

**Aturan "tidak ada localStorage" di `index.html` TIDAK berlaku di sini.** Satu grid
12×12 berarti 144 panggilan API berbayar; kehilangan itu karena tab tertutup tidak
bisa diterima. Snapshot disimpan ke IndexedDB (`ai-influencer-studio` → `snapshot`
→ slot `autosave`) dan ditawarkan lewat banner pemulihan saat app dibuka lagi.

- **API key tetap tidak pernah menyentuh disk.** `snapshotForDisk()` melewatkan
  `state.api` ke penyaring `/key$/i` yang sama seperti ekspor. Jangan longgarkan.
- `autosaveSoon()` menjarangkan diri sendiri berdasarkan `imageWeight()` — 4 s saat
  ringan, 30 s saat gambar lewat 40 MB, karena men-serialize ratusan MB tiap
  beberapa detik akan membekukan UI.
- Kuota penuh → otomatis turun ke snapshot **tanpa gambar** supaya rencana dan teks
  tetap selamat, dan banner pemulihan menandainya (`_tanpaGambar`).
- IndexedDB diblokir (mode privat) → `dbFailed` menyetel diam, jangan munculkan
  error berulang.
- `normalizeState()` dipakai bersama oleh `loadProject` dan pemulihan autosave —
  satu tempat untuk menambal bentuk state lama.

### Gerbang mutu otomatis

`state.qaGate` = `{on, min, retry}`. Saat aktif, `renderShot()` memeriksa tiap hasil
lewat `qaImage()`; kalau di bawah `min`, prompt diulang dengan `FIX: <saran QA>` dan
**hanya percobaan berskor tertinggi yang disimpan**. Sengaja **tidak** dipasang di
`renderGridCell()` — 144 sel × (render + QA) akan melipatgandakan biaya.

`state.usage` = `{img, txt, harga}` menghitung panggilan API per proyek. `harga`
diisi user, bukan ditebak aplikasi, karena tiap gateway berbeda.

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

### Ekspor

Tiap artefak punya fungsi **penghasil isi** terpisah dari fungsi pengunduh —
`promptsText()`, `contentText()`, `mediaKitText()`, `sheetHTML()`, `kalenderCSV()`,
`qaCSV()`, `projectJSON()`. Arsip ZIP memakai fungsi yang sama persis, jadi isi file
tunggal dan isi arsip tidak mungkin berbeda. Kalau menambah artefak baru, pisahkan
juga dan daftarkan di `exportZip()`.

`makeZip()` menulis ZIP sendiri (CRC32 + local header + central directory + EOCD)
karena app ini satu file dan dipakai offline — **jangan tarik JSZip dari CDN**.
Gambar disimpan `compress:false` (PNG sudah terkompresi), teks dideflate lewat
`CompressionStream('deflate-raw')` bila tersedia. Zip64 tidak didukung: >65535 file
atau >4 GB dilempar sebagai error yang bisa dibaca user.

## Verifikasi perubahan

Tidak ada test runner. Buka di browser, cek console bersih, lalu smoke test.

- `index.html`: setup → input → generate satu tahap AI → ekspor Remotion → cek
  JSON-nya valid.
- `influencer.html`: isi key → upload foto → generate identitas → susun rencana
  foto → render → ekspor project, lalu buka file .json-nya dan pastikan tidak ada
  API key di dalamnya.

Satu error console yang wajar muncul di kedua file saat offline: stylesheet Google
Fonts gagal dimuat. Font jatuh ke `system-ui`, bukan bug.
