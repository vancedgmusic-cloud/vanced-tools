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

## Keamanan

API key ada di `state.api`, memori saja. `saveProject()` (baris ~1862) menghapusnya
dari file ekspor lewat **blocklist hardcoded**:

```js
snap.api={...snap.api, claudeKey:'',geminiKey:'',groqKey:'',orKey:'',customKey:''};
```

Kalau menambah provider baru dengan field key baru, **wajib tambahkan ke blocklist
itu** — kalau tidak, key ikut ter-download ke file project user. `loadProject` juga
sengaja mempertahankan `state.api` yang sedang aktif dan tidak menimpanya dari file.

Key hanya dikirim ke API resmi provider-nya. Jangan tambah endpoint pihak ketiga,
telemetry, atau logging yang menyentuh key.

## Verifikasi perubahan

Tidak ada test runner. Buka di browser, cek console bersih, lalu smoke test:
setup → input → generate satu tahap AI → ekspor Remotion → cek JSON-nya valid.
