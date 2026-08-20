# Vanced Tools

Dua single-file browser app, UI Bahasa Indonesia, tanpa build step:

| File | Isi |
|---|---|
| `index.html` | Framework Brain Director — pipeline riset produk → storyboard → props Remotion |
| `voice-clone.html` | Voice Clone Studio — sample suara → voice clone → text to speech |

Keduanya berbagi konvensi yang sama: satu file `<style>` + markup + `<script>`,
`'use strict'`, state global di memori, `data-path` untuk binding form, `toast()`
untuk feedback, dan penghapusan API key berdasarkan bentuk nama field saat ekspor
project. Baca bagian **Keamanan** di bawah sebelum menambah provider di app mana pun.

# Framework Brain Director

Single-file browser app (`index.html`, ~1.9k lines). Bahasa Indonesia UI.
Pipeline riset produk → storyboard → video prompt → ekspor props Remotion.

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

# Voice Clone Studio

Single-file browser app (`voice-clone.html`). Alur `STEPS`:
`sample → clone → tts → library`.

Kloning suara **tidak terjadi di browser** — tidak ada model TTS lokal yang bisa
meniru suara dari sample. App ini klien untuk penyedia voice cloning memakai API
key user sendiri. Jangan mencoba "menambahkan clone offline"; yang bisa ditambah
hanyalah penyedia baru.

## Adapter penyedia

`ADAPTERS` (bukan `callAI` seperti di `index.html`) — tiap penyedia mengisi kontrak
yang sama: `test`, `listVoices`, `clone`, `tts`, `del`. Menambah penyedia =
satu entri di `ADAPTERS` + satu di `PROVIDERS`. Yang sudah ada:

- `eleven` — ElevenLabs. `POST /v1/voices/add` (multipart) → `voice_id`,
  lalu `POST /v1/text-to-speech/{id}` → audio biner.
- `fish` — Fish Audio. `POST /model` (multipart) → `_id`, lalu `POST /v1/tts`.
  Sering diblokir CORS dari browser; pesan errornya sudah menyebut itu.
- `openai` — gateway OpenAI-compatible mana pun. `POST {base}/audio/speech`.
  Base URL milik user; `base()` menormalkan akhiran `/v1` supaya "https://host"
  dan "https://host/v1" sama-sama jalan. Daftar suaranya diketik user di
  `api.oaVoiceList` karena API ini tidak mengumumkan suara yang tersedia.
  `fetchOaModels()` menarik `GET {base}/models` lalu menyaringnya dengan
  `TTS_LOOKING`; nol hasil = gateway itu kemungkinan tidak melayani TTS.
  Base URL milik user (dikonfirmasi): Dinoiki `https://ai.dinoiki.com/v1`,
  Koboi LLM Lite `https://lite.koboillm.com/v1`. Menurut `AI_Talent_Forge.html`
  di branch `claude/ai-influencer-template-ugc-g5fzis`, **Dinoiki menyediakan
  `/audio/speech`, Koboi tidak** — karena itu HTTP 404 di endpoint ini
  diterjemahkan jadi pesan tersendiri, bukan error mentah.
- `local` — Web Speech API. **Tidak mengkloning apa pun** dan `tts()`-nya
  mengembalikan `null` (tidak ada blob). Jangan bikin kode yang mengasumsikan
  `tts()` selalu memberi Blob.

`PROVIDERS[].clone` menandai penyedia yang benar-benar bisa mengkloning. `openai`
dan `local` ber-`clone:false`: langkah 02 menolaknya (di `cloneVoice()`, bukan cuma
tombol disabled), tombol hapus suara disembunyikan lewat `canClone()`, dan
`saveProject` tidak menyimpan suaranya karena selalu bisa dibangun ulang.
**Jangan menambah penyedia non-cloning dengan `clone:true`** — langkah 02 akan
mengirim sample ke endpoint yang tidak ada.

`req(url, opts, label, want)` menangani retry transien (429/5xx) 3x dengan backoff,
sama semangatnya dengan `postJSON` di `index.html`. `want` = `json` | `blob` | `none`.

## Aturan domain (jangan dilanggar)

- **Gate izin.** `cloneVoice()` menolak jalan kalau `state.consent` false, dan cek
  itu ada **sebelum** `guard()` — bukan hanya lewat tombol yang `disabled`, supaya
  tidak bisa dilewati dari console atau lewat project yang dimuat. Urutannya juga
  disengaja: `guard()` membuka modal API kalau key kosong, dan modal itu menutupi
  checkbox izinnya. Jangan tukar urutannya, jangan pindahkan gate ke UI saja.
- Sample audio (`state.samples[].blob`) **tidak ikut** ke file project — ukurannya
  bisa ratusan MB. Yang dipertahankan adalah `voices` (voice id di sisi penyedia).
  `loadProject` karena itu tidak me-restore `samples`, dan menandai `history` lama
  sebagai tanpa blob.
- Voice hidup di akun penyedia, bukan di file ini. `deleteVoice()` menghapusnya
  permanen di sana — pertahankan `confirm()`-nya.

## Analisis sample

`analyze(blob)` memakai `decodeAudioData` untuk durasi, peaks (waveform), level
puncak, dan rasio hening. Dipakai untuk memperingatkan clipping / volume terlalu
pelan / terlalu banyak diam sebelum sample dikirim. Format yang gagal didekode
tetap boleh dipakai — durasinya diambil dari elemen `<audio>`, tanpa waveform.
`toWav()` mengubah hasil `MediaRecorder` (webm/mp4) jadi WAV 16-bit mono karena
tidak semua penyedia menerima webm.

Teks panjang dipotong `chunkText()` per `CHUNK_CHARS` (2400) di batas kalimat, lalu
blob MP3-nya dirangkai jadi satu file. Untuk ElevenLabs tiap potongan mengirim
`previous_text` / `next_text` supaya prosodi antar-potongan nyambung.

Perangkaian itu **hanya sah untuk MP3** karena frame MP3 bisa dideret. WAV/FLAC/AAC
punya header di depan berkas, jadi `speak()` menolak kombinasi >1 potongan + format
non-MP3 (lihat `outFormat()`) daripada menghasilkan file rusak yang tetap terunduh.

## Keamanan

Berlaku untuk kedua app. API key ada di `state.api`, memori saja. `saveProject()`
menghapusnya dari file ekspor berdasarkan **bentuk nama field**, bukan daftar nama:

```js
snap.api=Object.fromEntries(Object.entries(snap.api).map(([k,v])=>[k, /key$/i.test(k)?'':v]));
```

Artinya provider baru aman secara default — asal field kredensialnya diberi nama
berakhiran `Key` (`deepseekKey`, `mistralKey`, `elevenKey`, `fishKey`, dst).
**Jangan menamai field kredensial di luar pola itu**, karena ia akan lolos ke file
project yang di-download dan dibagikan user. `loadProject` di kedua app juga sengaja
mempertahankan `state.api` yang sedang aktif dan tidak menimpanya dari file.

Key hanya dikirim ke API resmi provider-nya. Jangan tambah endpoint pihak ketiga,
telemetry, atau logging yang menyentuh key.

## Verifikasi perubahan

Tidak ada test runner. Buka di browser, cek console bersih, lalu smoke test.

`index.html`: setup → input → generate satu tahap AI → ekspor Remotion → cek JSON-nya valid.

`voice-clone.html`: unggah sample → cek waveform & flag kualitas muncul → centang izin
→ buat clone → TTS → audio bisa diputar & diunduh → Simpan, lalu pastikan `api.*Key`
di file JSON-nya kosong. Tanpa API key berbayar, jalur ini bisa diuji dengan
mem-*intercept* request penyedia (mis. `page.route` di Playwright) — Chromium sudah
tersedia di environment ini.
