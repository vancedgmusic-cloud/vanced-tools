# vanced-tools

Kumpulan project HTML — setiap tool adalah satu file HTML mandiri, tanpa build step, tanpa server. Cukup buka di browser.

## Tool

### `AI_Talent_Forge.html` — Pembuat AI Influencer
Membangun AI influencer / virtual talent dari nol sampai siap produksi: persona, cetak biru wajah, wardrobe, suara, prompt gambar, prompt video, mesin konten, dan aturan pengungkapan AI.

Hasil akhirnya adalah **Talent Block** yang ditempel ke Framework Brain Director, sehingga talent yang sama dipakai lintas kedua tool.

**13 tahap:**

| Tahap | Isi |
|---|---|
| 00 Setup Talent | Gender, usia, etnis, pasar, platform, engine gambar/video, rasio |
| 01 Brief & Niche | Niche, kategori produk, audiens, arketipe |
| 02 Referensi Visual | Unggah acuan (Pinterest/IG), AI baca jadi deskripsi fisik + **pembeda** |
| 03 Persona Core | Nama, backstory, kepribadian, tone of voice, pantangan |
| 04 Character DNA | Cetak biru fisik + **Identity Lock** & **Negative Lock** |
| 05 Realisme | Kontrol anti-slop → **Realism Lock** + negative + parameter engine |
| 06 Wardrobe & Style | Signature look, capsule wardrobe, palet warna, **Wardrobe Lock** |
| 07 Voice & Speech | Karakter suara, filler words, catchphrase, voice design prompt |
| 08 Reference Sheet | 10 prompt: turnaround 4 sudut, ekspresi, close-up, full body, tangan |
| 09 Scene Pack | Scene UGC/affiliate siap produksi + prompt gambar, video, dialog |
| 10 Content Engine | Positioning, content pillars, bank ide Shorts & longform, monetisasi |
| 11 Compliance | Disclosure AI + afiliasi, aturan klaim, checklist sebelum posting |
| 12 Audit & Ekspor | 25 pemeriksaan konsistensi, realisme & struktur, Talent Block, ekspor MD/JSON |

**Referensi visual.** Unggah sampai 4 gambar acuan dengan peran masing-masing (wajah, tubuh, gaya, mood). AI membacanya jadi deskripsi fisik terstruktur, lalu Character DNA diturunkan dari bacaan itu — bukan dikarang dari nol. Gambar dikecilkan otomatis ke 768px sehingga ikut tersimpan di project.

Mode **Komposit** (default) menurunkan ciri fisik tapi mewajibkan AI mengusulkan **pembeda** konkret, supaya hasil akhirnya orang baru dan bukan salinan orang di foto. Meniru wajah orang nyata yang bisa dikenali menyentuh hak atas potret dan bisa berujung penangguhan akun. Mode **Inspirasi** hanya mengambil vibe dan gaya.

**Realisme (anti AI-slop).** Detail halus kulit — pori, bulu vellus, variasi pigmen — secara statistik mirip noise, jadi ikut terbuang di tahap awal diffusion; ditambah lagi model banyak belajar dari foto yang sudah diretouch. Artinya tekstur nyata **tidak muncul sendiri, harus diminta eksplisit**.

Tahap 05 menyusun **Realism Lock** dari 8 kontrol (tekstur kulit, garis & kerutan, ketidaksempurnaan, kilap, realisme tubuh, kamera & lensa, pencahayaan, grain). Blok itu ditempel ke setiap prompt bersama Identity Lock, dilengkapi:
- **Negative anti-slop** — menolak `plastic skin`, `airbrushed`, `poreless`, `waxy`, `beauty filter`, `doll-like`, dst.
- **Daftar kata terlarang** — `flawless`, `perfect skin`, `beautiful`, `8k`, `masterpiece` justru menarik hasil ke arah plastik, jadi AI dilarang memakainya.
- **Parameter engine** — Midjourney `--style raw --s <nilai>` (wajib, tanpa itu gaya bawaan bikin orang terlihat seperti lukisan), Flux `guidance 1.5–2.0`, SDXL skin LoRA + ADetailer.

Panjang blok menyesuaikan engine: versi penuh (~275 kata) untuk engine bahasa natural yang makin patuh dengan deskripsi panjang, versi ringkas (~45 kata) untuk Midjourney/SDXL yang memberi bobot per frasa.

Detail yang paling sering gagal diminta eksplisit: iris + limbal ring + catchlight yang cocok arah cahaya, rambut dengan helai lepas, gigi dengan ketidakteraturan wajar, dan proporsi tubuh dengan lipatan kulit alami.

**Realisme gerak (khusus video).** Realism Lock di atas semuanya soal kulit yang diam. Di video yang bikin hasil terasa palsu justru fisika, jadi ada **Motion Lock** terpisah yang hanya ditempel ke prompt video: berat & inersia, akselerasi natural, rambut dan kain telat sepersekian detik dari badan, kaki menapak tanpa sliding, motion blur sesuai shutter, parallax konsisten kedalaman, plus micro-shake dan rolling-shutter wobble.

**Struktur BEATS.** Satu shot pendek tetap butuh tiga titik, kalau tidak model cenderung menghasilkan gerakan datar tanpa perkembangan. Setiap scene wajib punya timeline bertimestamp yang skalanya ikut durasi — untuk 8 detik jadi `t=0–1.4s HOOK · t=1.4–5.2s BUILD · t=5.2–8s PAYOFF`.

**Slot komposisi & aksi.** Tiap scene wajib punya kata kerja fisik yang terbaca instan (bukan "berpose") dan komposisi eksplisit — sudut, jarak, aturan framing. Minimal 2 scene wajib menyediakan **negative space** untuk teks overlay, karena caption dan sticker butuh ruang yang direncanakan, bukan sisa.

**Kontinuitas gambar → video.** Prompt video wajib mewarisi subjek, wardrobe, setting, dan komposisi dari prompt gambar scene yang sama. Yang boleh ditambahkan hanya gerakan kamera, perkembangan aksi sesuai beats, perilaku partikel, dan realisme gerak.

**Cara kerja konsistensi wajah.** Tahap 04 menghasilkan satu baris padat berisi ciri paling mengunci (Identity Lock). Baris itu ditempel otomatis di depan **setiap** prompt gambar dan video, diperkuat Negative Lock (`do not change: ...`) dan Wardrobe Lock. Audit di tahap 12 memverifikasi setiap prompt benar-benar memuatnya — termasuk memeriksa penanda tekstur kulit, detail mata, dan ada-tidaknya kata pemicu slop.

**Engine yang didukung** — sintaks prompt menyesuaikan otomatis:
- Gambar: Flux.2, Midjourney v7 (`--cref`/`--sref`), SDXL/Pony, Nano Banana, Seedream 4, Qwen-Image, Ideogram v3
- Video: Veo 3.1, Kling 3.0, Seedance 2.0, Runway Gen-4, Wan 2.5, Hailuo

**Provider AI:** Claude (bawaan Claude.ai / API key), Gemini, Groq, OpenRouter, atau endpoint OpenAI-compatible mana pun.

### `Framework_Brain_Director_Fixed (8).html` — Creative Director Video Affiliate
Merakit storyboard dan prompt video affiliate per produk: analisis produk → marketing → content angle → creative brief → storyboard → prompt video → caption & hashtag.

## Alur gabungan

```
AI_Talent_Forge ──► Talent Block ──► Framework_Brain_Director
   (siapa talentnya)                     (mau jualan apa)
```

Bangun talent sekali, pakai untuk semua produk.

## Catatan teknis

- **API key tidak pernah disimpan** — hanya hidup di memori tab, tidak ikut ke localStorage maupun file `.json`.
- **Gambar referensi dikecilkan ke 768px** sebelum masuk penyimpanan, jadi ikut tersimpan di project tanpa menjebol kuota localStorage — dan otomatis ikut ke file `.json`, jadi hati-hati kalau file project dibagikan.
- Autosave ke localStorage; ekspor `.json` untuk cadangan dan pindah perangkat.
- Semua provider punya retry otomatis saat server sibuk, dan loop kontinuasi supaya output panjang tidak terpotong.

## Referensi

Riset yang mendasari struktur tool ini:

- [Consistent AI influencer workflow](https://higgsfield.ai/blog/how-to-create-ai-influencer) · [reference sheet & Character DNA](https://opencreator.io/blog/ai-character-reference-sheet) · [prompt turnaround dan kunci seed](https://consistentcharacterai.pro/blog/ai-character-reference-sheet-prompt-guide)
- [InstantID](https://github.com/instantX-research/InstantID) (identity-preserving, zero-shot) · [AI-Influencer-Generator](https://github.com/SamurAIGPT/AI-Influencer-Generator) (MIT, SD + TTS + SadTalker) — jalur alternatif tanpa training LoRA
- [Perbandingan model video 2026](https://aimlapi.com/blog/best-ai-video-generators-2026-veo-3-1-kling-sora-2-seedance-more-compared) · [panduan Veo 3.1](https://www.versely.studio/blog/veo-3-1-complete-guide-google-ai-video-model) · [panduan prompting Kling 3](https://oakgen.ai/blog/kling-3-prompting-guide)
- [Framework script UGC: Hook → Problem → Solution → CTA](https://ckstudio.in/ultimate-ugc-video-script-framework-hook-problem-product-cta/) · [strategi hook & A/B testing](https://alici.ai/blog/ugc-hook-strategy-types-testing-2026)
- [ElevenLabs Voice Design](https://elevenlabs.io/docs/eleven-creative/voices/voice-design) · [panduan prompting v3](https://elevenlabsmagazine.com/elevenlabs-voice-design-guide-2026/)
- [Aturan disclosure AI influencer](https://www.auditsocials.com/blog/ai-generated-influencer-content-compliance-disclosure-rules-2026) · [evaluasi FTC atas konten influencer AI](https://www.disclosurefacts.com/blog/why-ai-generated-influencer-content-may-violate-ftc-rules)
- [Prompt Engineering Masterclass](docs/prompt-engineering-masterclass.md) (dokumen internal) — master formula IMAGE/VIDEO. Yang diambil: slot BEATS bertimestamp, physics gerak, komposisi eksplisit + negative space, dan aturan kontinuitas still→video. Catatan: contoh prompt di dokumen itu memakai `ultra-realistic` / `hyper-realistic` yang justru masuk daftar kata terlarang di tool ini — yang bekerja pada contoh tersebut adalah detail spesifiknya, bukan buzzword-nya.
- [Strategi Shorts vs longform](https://influenceflow.io/resources/youtube-shorts-and-long-form-video-strategy-the-complete-2026-creators-guide-1/) · [monetisasi Shorts 2026](https://www.ssemble.com/blog/youtube-shorts-monetization-guide-2026)

Khusus realisme & anti-slop:

- [Kenapa kulit AI terlihat plastik dan cara memperbaikinya](https://medium.com/ai-analytics-diaries/your-ai-photos-look-fake-heres-how-to-fix-plastic-looking-skin-746202ceb7de) · [membuat tekstur kulit realistis](https://thinkpeak.ai/creating-realistic-skin-texture-ai-art/) · [memperbaiki tekstur kulit AI](https://www.wearview.co/blog/fix-ai-skin-texture)
- [Prompt potret realistis FLUX.1](https://www.nextdiffusion.ai/blogs/mastering-ai-portrait-prompts-with-flux1-for-realistic-images) · [gaya foto amatir & guidance rendah](https://ageofllms.com/ai-howto-prompts/ai-fun/flux-selfie-prompts) · [arahan seni FLUX & Midjourney](https://blog.designhero.tv/ai-art-direction-prompts-flux-midjourney/)
- [Midjourney v7: `--style raw` dan stylize rendah](https://www.digitbin.com/midjourney-v7-prompts-realistic-photos/) · [panduan potret realistis Midjourney](https://www.theklaystudio.com/midjourney-realistic-portraits-complete-guide-to-lifelike-ai-art/)
- [Hak atas potret pada AI influencer](https://journals.library.columbia.edu/index.php/lawandarts/article/download/14632/8019) · [aturan kemiripan wajah 2026](https://www.influencers-time.com/ai-likeness-rules-2026-disclosure-guide-for-marketers/)
