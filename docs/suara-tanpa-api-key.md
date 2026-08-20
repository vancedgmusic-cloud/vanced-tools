# Suara Clone Tanpa API Key

Panduan menjalankan voice cloning di komputer sendiri, supaya suara hasil clone
bisa dipakai terus-menerus: tanpa API key, tanpa kuota, tanpa biaya bulanan,
tanpa internet.

Ini pasangan dari `voice-clone.html`. Setelah server lokalnya jalan, aplikasi itu
tinggal diarahkan ke `http://localhost:8004/v1` lewat tombol **⚙ API**.

---

## Kenapa Chatterbox, bukan yang lain

Empat kandidat serius dibandingkan untuk keperluan voice over Bahasa Indonesia:

| Model | Bahasa Indonesia | Server siap pakai | Jalan tanpa GPU |
|---|---|---|---|
| **Chatterbox Multilingual** | Lewat **Melayu** | ✅ endpoint OpenAI-compatible | ✅ |
| XTTS-v2 (Coqui) | ❌ tidak ada di 17 bahasanya | perlu server pihak ketiga | ✅ tapi lambat |
| OpenAudio S1-mini | ❌ (Inggris, Mandarin, Jepang) | ❌ | ❌ |
| IndexTTS | ❌ (Mandarin, Inggris) | ❌ | ❌ |

Chatterbox menang karena tiga hal yang menentukan di praktik:

1. **Server resminya sudah bicara "bahasa" yang sama dengan aplikasi kita.**
   [`devnen/Chatterbox-TTS-Server`](https://github.com/devnen/Chatterbox-TTS-Server)
   menyajikan `POST /v1/audio/speech` — persis kontrak yang dipakai adapter
   *Gateway OpenAI-compatible*. Tidak ada kode baru yang perlu ditulis.
2. **Tidak minta API key sama sekali.** Servernya jalan tanpa autentikasi.
3. **Ada jalur CPU.** Kalau komputermu tidak punya GPU NVIDIA, tetap jalan —
   cuma lebih lambat.

### Jujur soal Bahasa Indonesia

Chatterbox **tidak mencantumkan Indonesia** di 23 bahasanya. Yang ada **Melayu**.

Dalam praktik ini biasanya cukup: Melayu dan Indonesia serumpun, ejaannya sama-sama
fonetis, dan pelafalan huruf per hurufnya nyaris identik. Yang perlu kamu waspadai
adalah **intonasi** yang kadang terasa Melayu, dan sebagian kosakata serapan yang
tekanannya bergeser.

Uji dulu dengan satu paragraf naskahmu yang sebenarnya sebelum memutuskan pindah
total. Kalau hasilnya belum sesuai selera, **Fish Audio** (yang sudah ada di
aplikasi) mendukung Bahasa Indonesia secara resmi dan kualitasnya jadi patokan —
konsekuensinya tetap butuh API key.

---

## Yang perlu disiapkan

- **Python 3.10** — versi ini spesifik. Python 3.11+ belum punya paket siap pakai
  untuk sebagian dependensinya.
- **Ruang disk** ±10 GB untuk bobot model.
- **GPU NVIDIA** kalau ada. Tidak wajib — ada jalur CPU, AMD ROCm, dan Apple Silicon.

Perkiraan kecepatan untuk satu kalimat pendek: beberapa detik di GPU, sekitar
setengah sampai satu menit di CPU. Untuk voice over 30 detik di CPU, siapkan
kesabaran beberapa menit — tapi tetap nol rupiah.

---

## Pemasangan

```bash
git clone https://github.com/devnen/Chatterbox-TTS-Server.git
cd Chatterbox-TTS-Server
```

Lalu jalankan launcher-nya — dia mendeteksi perangkat kerasmu dan memasang
dependensi yang cocok sendiri:

```bash
./start.sh          # Linux / macOS
start.bat           # Windows
```

Unduhan model berjalan otomatis saat pertama kali dijalankan, jadi proses pertama
memang lama. Setelah selesai, buka `http://localhost:8004` di browser untuk
memastikan Web UI-nya muncul.

---

## Menyiapkan suaramu

Di Chatterbox tidak ada langkah "kirim sample lalu dapat voice id". Suaranya
dikloning **saat itu juga** dari berkas referensi. Alurnya jadi lebih sederhana:

1. Buka `voice-clone.html`, masuk **langkah 01**, unggah atau rekam sample suaramu.
   Perhatikan flag kualitasnya — clipping dan hening berlebih tetap merusak hasil.
2. Klik tombol **⬇** di kartu sample untuk mengunduhnya.
3. Pindahkan berkas itu ke folder `reference_audio/` di dalam folder server.
   Beri nama yang gampang diingat, mis. `suara-saya.wav`.
4. **Langkah 02 dilewati** — tidak ada yang perlu dikirim ke mana pun.

Sample 10–20 detik yang bersih sudah cukup. Lebih panjang tidak otomatis lebih baik.

---

## Menghubungkan ke aplikasi

Di `voice-clone.html`, klik **⚙ API**:

| Kolom | Isi |
|---|---|
| Penyedia | **Gateway OpenAI-compatible** |
| Base URL | tombol preset **Server lokal (Chatterbox)** |
| API Key | isi apa saja, mis. `lokal` — server tidak memeriksanya |
| Daftar suara | nama berkas referensimu, mis. `suara-saya.wav` |
| Model | biarkan apa adanya — server lokal mengabaikannya |

Klik **Tes koneksi**. Kalau terhubung, lanjut ke **langkah 03**, pilih suaramu,
tulis naskah, tekan **🔊 Hasilkan suara**.

Mulai sekarang tidak ada kuota yang berkurang, berapa kali pun kamu generate.

---

## Kalau macet

**"Permintaan tidak sampai" / diblokir CORS.**
Halaman dibuka dari `file://` sementara server ada di `localhost` — browser
menganggapnya lintas-asal. Aktifkan CORS di konfigurasi server (izinkan origin
`*`), lalu jalankan ulang servernya.

**Server jalan tapi TTS balas 404.**
Endpoint `/v1/audio/speech` belum aktif. Cek base URL-nya sudah berakhiran `/v1`,
dan pastikan versi server yang kamu pasang memang punya endpoint OpenAI-compatible.

**Suaranya terdengar Melayu.**
Ini batas modelnya, bukan salah setelan. Yang bisa menolong: pilih sample referensi
dengan logat Indonesia yang kuat, dan tulis naskah dengan ejaan yang mengarahkan
pelafalan. Kalau tetap tidak sesuai, kembali ke Fish Audio untuk pekerjaan yang
menuntut logat presisi.

**Terlalu lambat di CPU.**
Potong naskah jadi bagian-bagian pendek dan generate satu per satu. Atau siapkan
audionya jauh hari — hasilnya berupa berkas MP3 yang jadi milikmu selamanya.

---

## Ringkasan pilihan

| Kebutuhan | Pakai |
|---|---|
| Volume besar, biaya nol, tanpa key | **Chatterbox lokal** |
| Logat Indonesia presisi, hasil paling mirip | ElevenLabs / Fish Audio |
| Sekadar mengecek naskah sebelum produksi | Mode Preview lokal di aplikasi |

Ketiganya bisa dipakai bergantian di aplikasi yang sama — tinggal ganti penyedia
di **⚙ API**. Tidak perlu memilih satu selamanya.
