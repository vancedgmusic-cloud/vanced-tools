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

Pipeline: `talent → identity → persona → shoot → render → content → export`.
Bikin virtual influencer dari foto talent: baca fotonya, kunci identitas, susun
persona, rencanakan sesi foto, render gambarnya, tulis caption.

**Dua mesin AI terpisah** di `state.api`, dipilih sendiri-sendiri:

- `api.txt` — teks & vision (`gemini`, `claude`, `dinoiki`, `koboillm`, `compat`).
  Dinoiki & KoboiLLM adalah gateway/reseller Indonesia; base URL-nya **selalu bisa
  diedit user**, jangan dikunci — key yang mereka jual bisa key upstream asli
  (OpenAI/Anthropic/Gemini) atau key gateway sendiri.
- `api.img` — gambar (`gemini_image`, `openai_image`, `compat_image`, `manual`).
  `gemini_image` menerima foto referensi langsung; `openai_image` pakai
  `/images/edits` multipart kalau ada referensi **dan** model cocok `/gpt-image/i`,
  selain itu `/images/generations`.

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
project kecuali user mencentang toggle-nya (`withImages`).

## Verifikasi perubahan

Tidak ada test runner. Buka di browser, cek console bersih, lalu smoke test.

- `index.html`: setup → input → generate satu tahap AI → ekspor Remotion → cek
  JSON-nya valid.
- `influencer.html`: isi key → upload foto → generate identitas → susun rencana
  foto → render → ekspor project, lalu buka file .json-nya dan pastikan tidak ada
  API key di dalamnya.

Satu error console yang wajar muncul di kedua file saat offline: stylesheet Google
Fonts gagal dimuat. Font jatuh ke `system-ui`, bukan bug.
