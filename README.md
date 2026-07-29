# vanced-tools

Kumpulan project HTML — setiap tool adalah satu file HTML mandiri, tanpa build step, tanpa server. Cukup buka di browser.

## Tool

### `AI_Talent_Forge.html` — Pembuat AI Influencer
Membangun AI influencer / virtual talent dari nol sampai siap produksi: persona, cetak biru wajah, wardrobe, suara, prompt gambar, prompt video, mesin konten, dan aturan pengungkapan AI.

Hasil akhirnya adalah **Talent Block** yang ditempel ke Framework Brain Director, sehingga talent yang sama dipakai lintas kedua tool.

**11 tahap:**

| Tahap | Isi |
|---|---|
| 00 Setup Talent | Gender, usia, etnis, pasar, platform, engine gambar/video, rasio |
| 01 Brief & Niche | Niche, kategori produk, audiens, arketipe, wajah referensi (opsional) |
| 02 Persona Core | Nama, backstory, kepribadian, tone of voice, pantangan |
| 03 Character DNA | Cetak biru fisik + **Identity Lock** & **Negative Lock** |
| 04 Wardrobe & Style | Signature look, capsule wardrobe, palet warna, **Wardrobe Lock** |
| 05 Voice & Speech | Karakter suara, filler words, catchphrase, voice design prompt |
| 06 Reference Sheet | 10 prompt: turnaround 4 sudut, ekspresi, close-up, full body, tangan |
| 07 Scene Pack | Scene UGC/affiliate siap produksi + prompt gambar, video, dialog |
| 08 Content Engine | Positioning, content pillars, bank ide Shorts & longform, monetisasi |
| 09 Compliance | Disclosure AI + afiliasi, aturan klaim, checklist sebelum posting |
| 10 Audit & Ekspor | 16 pemeriksaan konsistensi, Talent Block, ekspor Markdown/JSON |

**Cara kerja konsistensi wajah.** Tahap 03 menghasilkan satu baris padat berisi ciri paling mengunci (Identity Lock). Baris itu ditempel otomatis di depan **setiap** prompt gambar dan video, diperkuat Negative Lock (`do not change: ...`) dan Wardrobe Lock. Audit di tahap 10 memverifikasi setiap prompt benar-benar memuatnya.

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
- **Gambar referensi wajah tidak ikut disimpan** — hanya dikirim ke provider AI saat generate. Yang permanen adalah Character DNA hasil bacaannya.
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
- [Strategi Shorts vs longform](https://influenceflow.io/resources/youtube-shorts-and-long-form-video-strategy-the-complete-2026-creators-guide-1/) · [monetisasi Shorts 2026](https://www.ssemble.com/blog/youtube-shorts-monetization-guide-2026)
