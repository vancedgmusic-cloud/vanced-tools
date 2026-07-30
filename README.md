# vanced-tools

Kumpulan project HTML — setiap tool adalah satu file HTML mandiri, tanpa build step, tanpa server. Cukup buka di browser.

## Tool

### `AI_Talent_Forge.html` — Pembuat AI Influencer
Membangun AI influencer / virtual talent dari nol sampai siap produksi: persona, cetak biru wajah, wardrobe, suara, prompt gambar, prompt video, mesin konten, dan aturan pengungkapan AI.

Hasil akhirnya adalah **Talent Block** yang ditempel ke Framework Brain Director, sehingga talent yang sama dipakai lintas kedua tool.

**15 tahap:**

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
| 10 Perbaikan & Seed | Prompt inpaint terarah per bagian gagal + strategi seed + catatan seed |
| 11 Content Engine | Positioning, content pillars, bank ide Shorts & longform, monetisasi |
| 12 Compliance | Disclosure AI + afiliasi, aturan klaim, checklist sebelum posting |
| 13 Audit & Ekspor | 42 pemeriksaan konsistensi, realisme, struktur & lipsync, Talent Block, ekspor MD/JSON |
| 14 Render Gambar | Jalankan prompt jadi gambar langsung di browser — **Pose Sheet 12 panel** + character sheet gabungan untuk image reference engine video |

**Referensi visual.** Unggah sampai 4 gambar acuan dengan peran masing-masing (wajah, tubuh, gaya, mood). AI membacanya jadi deskripsi fisik terstruktur, lalu Character DNA diturunkan dari bacaan itu — bukan dikarang dari nol. Gambar dikecilkan otomatis ke 768px sehingga ikut tersimpan di project.

Mode **Komposit** (default) menurunkan ciri fisik tapi mewajibkan AI mengusulkan **pembeda** konkret, supaya hasil akhirnya orang baru dan bukan salinan orang di foto. Meniru wajah orang nyata yang bisa dikenali menyentuh hak atas potret dan bisa berujung penangguhan akun. Mode **Inspirasi** hanya mengambil vibe dan gaya.

**Realisme (anti AI-slop).** Detail halus kulit — pori, bulu vellus, variasi pigmen — secara statistik mirip noise, jadi ikut terbuang di tahap awal diffusion; ditambah lagi model banyak belajar dari foto yang sudah diretouch. Artinya tekstur nyata **tidak muncul sendiri, harus diminta eksplisit**.

Tahap 05 menyusun **Realism Lock** dari 8 kontrol (tekstur kulit, garis & kerutan, ketidaksempurnaan, kilap, realisme tubuh, kamera & lensa, pencahayaan, grain). Blok itu ditempel ke setiap prompt bersama Identity Lock, dilengkapi:
- **Negative anti-slop** — menolak `plastic skin`, `airbrushed`, `poreless`, `waxy`, `beauty filter`, `doll-like`, dst.
- **Daftar kata terlarang** — `flawless`, `perfect skin`, `beautiful`, `8k`, `masterpiece` justru menarik hasil ke arah plastik, jadi AI dilarang memakainya.
- **Parameter engine** — Midjourney `--style raw --s <nilai>` (wajib, tanpa itu gaya bawaan bikin orang terlihat seperti lukisan), Flux `guidance 1.5–2.0`, SDXL skin LoRA + ADetailer.

Panjang blok menyesuaikan engine: versi penuh (~275 kata) untuk engine bahasa natural yang makin patuh dengan deskripsi panjang, versi ringkas (~45 kata) untuk Midjourney/SDXL yang memberi bobot per frasa.

Detail yang paling sering gagal diminta eksplisit: iris + limbal ring + catchlight yang cocok arah cahaya, rambut dengan helai lepas, gigi dengan ketidakteraturan wajar, dan proporsi tubuh dengan lipatan kulit alami.

**Realism Lock dipecah dua — dan ini bukan kosmetik.** *Blok A* berisi sifat subjek (kulit, mata, rambut, gigi, tubuh) yang berlaku di shot mana pun, jadi ditempel ke setiap prompt. *Blok B* berisi kondisi pengambilan (kamera, pencahayaan, grain) yang berbeda per shot — turnaround butuh studio rata, scene luar ruang punya cahaya matahari sendiri, orbit tidak mungkin pakai kamera selfie sepanjang tangan. Blok B adalah default yang boleh diganti, dan satu prompt hanya boleh memuat **satu** kamera dan **satu** sumber cahaya. Audit menolak yang menyebut dua.

Sebelum dipecah, keduanya digabung dan ditempel ke semua prompt, sehingga setiap prompt berisi dua kamera dan dua pencahayaan yang bertabrakan.

**Urutan prompt itu menentukan bobot.** Engine memberi bobot lebih besar pada yang di depan, jadi urutannya dipatok: Identity Lock → aksi & setting → wardrobe → komposisi → kamera & cahaya → blok realisme → larangan. Blok realisme adalah pengubah permukaan, tempatnya *setelah* isi shot. Menaruhnya di depan mendorong aksi sebenarnya ke ujung prompt dan melemahkan bobotnya — di output nyata aksi bisa terhimpit di 8% terakhir. Audit menolak prompt yang menaruh realisme sebelum komposisi.

**Nol kamera sama merusaknya dengan dua.** Kalau kamera tidak disebut, engine menebak sendiri dan hasil antar shot jadi tidak seragam. Audit memeriksa kedua arah — kelebihan maupun kekurangan — dan kata "camera" saja tidak dihitung karena muncul juga di kalimat seperti "looking at the camera".

**Realisme gerak (khusus video).** Realism Lock di atas semuanya soal kulit yang diam. Di video yang bikin hasil terasa palsu justru fisika, jadi ada **Motion Lock** terpisah yang hanya ditempel ke prompt video: berat & inersia, akselerasi natural, rambut dan kain telat sepersekian detik dari badan, kaki menapak tanpa sliding, motion blur sesuai shutter, parallax konsisten kedalaman, plus micro-shake dan rolling-shutter wobble.

**Struktur BEATS.** Satu shot pendek tetap butuh tiga titik, kalau tidak model cenderung menghasilkan gerakan datar tanpa perkembangan. Setiap scene wajib punya timeline bertimestamp yang skalanya ikut durasi — untuk 8 detik jadi `t=0–1.4s HOOK · t=1.4–5.2s BUILD · t=5.2–8s PAYOFF`.

**Slot komposisi & aksi.** Tiap scene wajib punya kata kerja fisik yang terbaca instan (bukan "berpose") dan komposisi eksplisit — sudut, jarak, aturan framing. Minimal 2 scene wajib menyediakan **negative space** untuk teks overlay, karena caption dan sticker butuh ruang yang direncanakan, bukan sisa.

**Disiplin gerak per shot.** Motion Lock menambahkan fisika, tapi *banyaknya* gerak harus beda per jenis shot. Untuk talking-head ber-lipsync, gerakan kepala berlebihan justru merusak sinkronisasi bibir dan langsung terbaca AI — di sana menahan diri lebih realistis daripada dinamis. Tiap scene dapat nilai `tenang` / `sedang` / `dinamis`, dan **setiap scene berdialog wajib `tenang`**. Audit menolak kalau dilanggar.

**Gerbang siap dianimasikan.** Frame yang diunggah sebagai *start frame* mengunci karakter jauh lebih kuat daripada teks prompt mana pun, jadi kualitasnya menentukan seluruh video. Reference Sheet menampilkan lima syarat kelulusan: wajah utuh tidak terpotong rapat, mata & mulut tajam tidak terhalang (syarat lipsync), tidak bergaya artistik, pencahayaan wajar, dan lolos sekilas pandang sebagai foto sungguhan.

**Pembobotan frasa per engine.** Sintaks penekanan tidak bisa ditukar antar engine, jadi tool menyesuaikannya otomatis. Midjourney memakai `::` dengan bobot relatif — dan **jumlah seluruh bobot wajib tetap positif**, kalau tidak promptnya ditolak; tool menjaga itu. SDXL memakai `(frasa:1.3)` dengan rentang aman 0,7–1,5, negatifnya ke field terpisah. Engine bahasa natural seperti Flux tidak punya sintaks bobot sama sekali — di sana penekanan lewat posisi frasa, dan tool mengatakannya terus terang alih-alih memalsukan sintaks yang tidak ada.

Catatan jujur untuk SDXL: penekanan di SDXL berpengaruh jauh lebih lemah daripada di SD1.5. LoRA kulit dan ADetailer lebih menentukan daripada angka bobot.

**Perbaikan terarah, bukan generate ulang.** Satu tangan rusak tidak layak dibayar dengan frame baru yang identitasnya sudah pas. Tahap 10 menghasilkan prompt inpaint untuk 6 titik gagal tersering (tangan, mata, gigi, kulit kehilangan tekstur, tepi rambut, teks label), masing-masing dengan denoise rekomendasi. Angka acuan: **denoise 0,35–0,45**; di 0,8 tambalan lepas dari pencahayaan aslinya. Prompt inpaint sengaja **tidak** memuat Identity Lock penuh — area masked terlalu kecil, dan menyebut ciri yang tidak terlihat di situ membuat model menggambar ulang hal yang salah.

**Strategi seed — dengan koreksi penting.** Seed adalah jangkar **tampilan**, bukan jangkar **identitas**. Seed sama dengan prompt berbeda langsung menyimpang begitu pose, cahaya, atau framing berubah. Jadi: kunci seed untuk 4 sudut turnaround (di sana hanya sudut yang berubah) dan saat menguji dua versi prompt; **bebaskan** seed untuk scene, karena mengunci di sana melawan variasi yang dibutuhkan tanpa memberi konsistensi wajah apa pun. Wajah tetap dikunci Identity Lock plus reference adapter. Audit memeriksa seed turnaround seragam.

**Lima format UGC.** Content Engine membedakan dua sumbu yang mudah tertukar: *babak di dalam satu video* (hook → demo → CTA) dan *jenis video* yang diproduksi berulang untuk mengisi kalender. Yang kedua kini eksplisit — UGC Entertainment, Street Interview, Unboxing, Product Review, ASMR — lengkap dengan contoh arah shot per format dan penanda mana yang berdialog. Tanpa pemisahan ini, bank ide gampang jadi variasi dari satu format saja.

**Anggaran kata per durasi.** Kepadatan bicara natural sekitar 2,5 kata/detik, jadi panjang dialog dihitung dari durasi klip — untuk 8 detik targetnya 14–23 kata, untuk 15 detik 27–44. Batas keras **15 detik per klip**; ide yang lebih panjang dipecah jadi dua scene `(1/2)` dan `(2/2)`. Audit menolak dialog yang melewati anggaran karena itu memaksa bicara cepat dan merusak lipsync.

**Tanpa teks tergambar di dalam video.** Teks hasil generate hampir selalu rusak, dan caption yang benar ditambahkan saat editing — bukan dibakar ke dalam frame. Setiap prompt wajib melarangnya, dan negative anti-slop menolak `subtitles`, `captions`, `lower-third`, serta `on-screen text`. Audit memeriksanya per scene.

**Kontinuitas gambar → video.** Prompt video wajib mewarisi subjek, wardrobe, setting, dan komposisi dari prompt gambar scene yang sama. Yang boleh ditambahkan hanya gerakan kamera, perkembangan aksi sesuai beats, perilaku partikel, dan realisme gerak.

**Cara kerja konsistensi wajah.** Tahap 04 menghasilkan satu baris padat berisi ciri paling mengunci (Identity Lock). Baris itu ditempel otomatis di depan **setiap** prompt gambar dan video, diperkuat Negative Lock (`do not change: ...`) dan Wardrobe Lock. Audit di tahap 13 memverifikasi setiap prompt benar-benar memuatnya — termasuk memeriksa penanda tekstur kulit, detail mata, dan ada-tidaknya kata pemicu slop.

**Engine yang didukung** — sintaks prompt menyesuaikan otomatis:
- Gambar: Flux.2, Midjourney v7 (`--cref`/`--sref`), SDXL/Pony, Nano Banana, GPT Image 2.0, Higgsfield Soul, Seedream 4, Qwen-Image, Ideogram v3
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

- **API key tidak disimpan secara default** — hanya hidup di memori tab. Ada opsi opt-in "Ingat key di perangkat ini"
  untuk pemakaian di ponsel; key lalu disimpan di slot localStorage terpisah, dan tetap TIDAK pernah ikut ke file
  project `.json`. Bisa dihapus lewat tombol "Lupakan key".
- **Bisa dipakai di ponsel** — rail tahap berubah jadi drawer di bawah 900px, dan setiap panel punya navigasi
  maju-mundur sendiri sehingga tidak ada tahap yang tanpa jalan keluar.
- **Gambar referensi dikecilkan ke 768px** sebelum masuk penyimpanan, jadi ikut tersimpan di project tanpa menjebol kuota localStorage — dan otomatis ikut ke file `.json`, jadi hati-hati kalau file project dibagikan.
- **Beberapa talent dalam satu browser** — tiap project punya slot localStorage sendiri, jadi memulai talent kedua
  tidak menimpa yang pertama. Menu Project bisa beralih, duplikat, hapus, ekspor, dan impor. Impor `.json`
  selalu masuk ke project baru, tidak pernah menimpa yang sedang dibuka.
- Autosave per project ke localStorage; ekspor `.json` untuk cadangan dan pindah perangkat.
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
- [Ultra-Realistic AI UGC Character Guide](docs/ultra-realistic-ugc-guide.md) (dokumen internal). Yang diambil: disiplin gerak per shot, gerbang siap-dianimasikan, alur start-frame, dan aturan dialog untuk lipsync. Panduan ini menyadarkan satu konflik nyata — prinsip "less movement = more realism" untuk talking-head berlawanan dengan dorongan sinematik dari masterclass, dan itu diselesaikan lewat disiplin gerak per shot alih-alih satu aturan untuk semua scene.
- [Magnific Content Factory skill](docs/magnific-content-factory-skill.md) (dokumen internal) — pipeline agen berbasis MCP, kategori berbeda dari tool ini. Yang diambil: taksonomi lima format UGC beserta contoh arah shot, tabel panjang naskah terhadap durasi, batas 15 detik per klip, dan aturan tanpa teks tergambar. Yang tidak diambil: seluruh lapisan orkestrasi MCP, penggerbangan kredit, penjadwalan Meta Ads, dan laporan biaya — semuanya menuntut akun berbayar dan konektor, sementara tool ini sengaja berjalan mandiri di browser.
- [Multi-prompt & weights Midjourney](https://docs.midjourney.com/hc/en-us/articles/32658968492557-Multi-Prompts-Weights) · [bobot prompt SDXL](https://apatero.com/blog/prompt-weighting-syntax-complete-guide-2025) · [ADetailer untuk wajah & tangan](https://stable-diffusion-art.com/adetailer/) · [seed Midjourney](https://docs.midjourney.com/hc/en-us/articles/32604356340877-Seeds) — seed adalah jangkar gaya, bukan jangkar identitas
- [Strategi Shorts vs longform](https://influenceflow.io/resources/youtube-shorts-and-long-form-video-strategy-the-complete-2026-creators-guide-1/) · [monetisasi Shorts 2026](https://www.ssemble.com/blog/youtube-shorts-monetization-guide-2026)

Khusus realisme & anti-slop:

- [Kenapa kulit AI terlihat plastik dan cara memperbaikinya](https://medium.com/ai-analytics-diaries/your-ai-photos-look-fake-heres-how-to-fix-plastic-looking-skin-746202ceb7de) · [membuat tekstur kulit realistis](https://thinkpeak.ai/creating-realistic-skin-texture-ai-art/) · [memperbaiki tekstur kulit AI](https://www.wearview.co/blog/fix-ai-skin-texture)
- [Prompt potret realistis FLUX.1](https://www.nextdiffusion.ai/blogs/mastering-ai-portrait-prompts-with-flux1-for-realistic-images) · [gaya foto amatir & guidance rendah](https://ageofllms.com/ai-howto-prompts/ai-fun/flux-selfie-prompts) · [arahan seni FLUX & Midjourney](https://blog.designhero.tv/ai-art-direction-prompts-flux-midjourney/)
- [Midjourney v7: `--style raw` dan stylize rendah](https://www.digitbin.com/midjourney-v7-prompts-realistic-photos/) · [panduan potret realistis Midjourney](https://www.theklaystudio.com/midjourney-realistic-portraits-complete-guide-to-lifelike-ai-art/)
- [Hak atas potret pada AI influencer](https://journals.library.columbia.edu/index.php/lawandarts/article/download/14632/8019) · [aturan kemiripan wajah 2026](https://www.influencers-time.com/ai-likeness-rules-2026-disclosure-guide-for-marketers/)
