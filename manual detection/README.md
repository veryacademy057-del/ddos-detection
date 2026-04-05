# 🌊 DDoS Attack Detection — Analisis Manual dengan Wireshark

> **Seri Lab:** Blue Team SOC Lab  
> **Platform:** [TryHackMe](https://tryhackme.com)  
> **YouTube:** [▶️ Tonton Tutorial di YouTube](https://youtu.be/E86twYGsINE?si=2kC3azBuhpivhH5_)  
> **Kategori:** `Network Analysis, Threat Detection`

---

## 📋 Deskripsi

Dokumentasi ini menjelaskan langkah-langkah mendeteksi serangan **Distributed Denial of Service (DDoS)** secara manual menggunakan **Wireshark** berdasarkan analisis file *packet capture* (PCAP). Analisis manual adalah kemampuan dasar yang wajib dimiliki setiap SOC Analyst sebelum beralih ke otomasi SIEM.

---

## 🎬 Video Tutorial

[![DDoS Detection - Manual Analysis with Wireshark](https://img.youtube.com/vi/E86twYGsINE/maxresdefault.jpg)](https://youtu.be/E86twYGsINE?si=2kC3azBuhpivhH5_)

> 📺 **[Tonton di YouTube → DDoS Attack Detection Manual Analysis](https://youtu.be/E86twYGsINE?si=2kC3azBuhpivhH5_)**

---

## 🎯 Tujuan Investigasi

- Mengidentifikasi adanya serangan DDoS dalam traffic jaringan
- Menemukan IP attacker yang mencurigakan
- Menentukan endpoint yang menjadi target serangan
- Menganalisis pola serangan (manual vs bot/script)

---

## 🛠️ Tools yang Digunakan

| Tool | Fungsi |
|------|--------|
| **Wireshark** | Analisis packet capture (PCAP) |
| **File PCAP** | Data traffic jaringan yang diinvestigasi |

---

## 🚨 Indikator Serangan DDoS

Sebelum mulai analisis, kenali tanda-tanda umum serangan DDoS:

| Indikator | Penjelasan |
|-----------|------------|
| Lonjakan traffic tiba-tiba | Volume paket naik drastis dalam waktu singkat |
| Request masif ke satu endpoint | Ribuan request ke `/login`, `/api`, atau halaman tertentu |
| Request berulang dalam waktu singkat | Pola otomatis — bukan perilaku pengguna normal |
| Banyak IP berbeda menyerang serentak | Ciri khas DDoS (distributed) bukan DoS biasa |

---

## 🔍 Langkah-Langkah Analisis

### Langkah 1 — Buka File PCAP

```
Buka Wireshark
→ File → Open
→ Pilih file .pcap
```

---

### Langkah 2 — Filter Traffic HTTP

Fokuskan tampilan hanya ke web traffic untuk mengurangi noise:

```
http
```

> 💡 Filter ini menyembunyikan protokol lain (DNS, ARP, dll) sehingga analisis lebih fokus ke traffic yang relevan.

---

### Langkah 3 — Identifikasi Lonjakan Traffic

Visualisasikan volume traffic dari waktu ke waktu:

```
Statistics → IO Graph
```

**Yang dicari:**

```
Jika grafik menunjukkan spike tajam yang tidak normal
→ Indikasi kuat adanya serangan
```

> 💡 Traffic normal biasanya stabil atau naik perlahan. Lonjakan vertikal yang tiba-tiba dalam hitungan detik adalah tanda serangan.

---

### Langkah 4 — Identifikasi Endpoint Target

Gunakan filter untuk melihat semua HTTP request:

```
http.request
```

Perhatikan kolom **Request URI** — cari endpoint yang muncul berulang-ulang:

| Endpoint Umum yang Diserang | Keterangan |
|-----------------------------|------------|
| `/login` | Login flooding |
| `/api` | API abuse |
| `/index` | Homepage flooding |
| `/search` | Search query flooding |

> 💡 Jika satu endpoint menerima ribuan request dalam waktu singkat, endpoint tersebut adalah target utama serangan.

---

### Langkah 5 — Identifikasi IP Attacker

**Cara 1 — Lewat packet langsung:**
```
Klik salah satu packet HTTP yang mencurigakan
→ Lihat field: Source IP
```

**Cara 2 — Lewat statistik koneksi:**
```
Statistics → Conversations → Tab IPv4
→ Sort berdasarkan kolom "Packets" (descending)
```

> 💡 IP dengan jumlah paket tertinggi adalah kandidat utama attacker. Pada serangan DDoS, akan terlihat **banyak IP berbeda** dengan jumlah request yang tidak wajar.

---

### Langkah 6 — Analisis Pola Request

Filter berdasarkan metode HTTP:

```
http.request.method == "GET"
```

Filter ke endpoint spesifik:

```
http.request.uri contains "login"
```

> 💡 Jika ratusan atau ribuan request ke endpoint yang sama dalam waktu singkat, ini bukan traffic normal.

---

### Langkah 7 — Follow TCP Stream

Lihat isi lengkap dari sebuah koneksi:

```
Klik kanan salah satu packet
→ Follow → TCP Stream
```

**Yang dicari:**
- Request yang identik berulang-ulang → tanda script/bot otomatis
- Header yang tidak lazim
- Tidak ada variasi dalam request → tidak seperti perilaku manusia

---

### Langkah 8 — Analisis User-Agent

User-Agent mengungkap identitas client yang mengirim request. Cari field ini di dalam packet HTTP dan waspadai nilai berikut:

| User-Agent | Keterangan |
|------------|------------|
| `curl/x.x.x` | Tool command line — bukan browser |
| `python-requests/x.x` | Script Python otomatis |
| `Mozilla/5.0 (identik semua)` | Bisa jadi spoofed |
| Kosong / tidak ada | Sangat mencurigakan |

> 💡 User manusia menggunakan browser dengan User-Agent yang beragam. Jika semua request menggunakan User-Agent yang sama persis atau User-Agent tool seperti `curl`/`python`, hampir pasti itu serangan otomatis.

---

### Langkah 9 — Konfirmasi Pola Serangan

Serangan DDoS terkonfirmasi jika ditemukan pola seperti:

```
Contoh 1:
1000+ request ke /login dalam < 1 menit dari 1 IP

Contoh 2:
500+ IP berbeda mengakses /api secara bersamaan dalam 30 detik
```

---

## ✅ Kriteria Kesimpulan Analisis

Serangan dikategorikan sebagai **DDoS** jika memenuhi kondisi berikut:

| Kriteria | Status |
|----------|--------|
| Terjadi lonjakan traffic signifikan di IO Graph | ✅ |
| Satu endpoint menjadi target utama request masif | ✅ |
| Request dilakukan secara berulang dan otomatis | ✅ |
| Ditemukan banyak IP mencurigakan dengan request tinggi | ✅ |

---

## 🛡️ Rekomendasi Mitigasi

### Level Aplikasi

| Mitigasi | Penjelasan |
|----------|------------|
| **CAPTCHA** | Membedakan manusia dan bot |
| **Rate Limiting** | Batasi jumlah request per IP per detik |
| **Validasi Input** | Tolak request dengan format tidak valid |

### Level Jaringan

| Mitigasi | Penjelasan |
|----------|------------|
| **CDN (Cloudflare)** | Distribusi traffic dan absorpsi serangan |
| **Web Application Firewall (WAF)** | Filter request berbahaya sebelum sampai ke server |
| **IP Blacklisting** | Blokir IP attacker yang teridentifikasi |

### Monitoring

| Mitigasi | Penjelasan |
|----------|------------|
| **SIEM (Splunk)** | Deteksi otomatis berbasis pola log |
| **Real-time Alerting** | Notifikasi instan saat anomali terdeteksi |

---

## 📌 Catatan Penting

> **Analisis manual** cocok untuk investigasi awal dan memahami pola serangan secara mendalam. Namun untuk skala traffic yang besar, **kombinasi manual + SIEM** memberikan hasil terbaik — manual untuk investigasi forensik, SIEM untuk deteksi real-time otomatis.

---

## 📚 Referensi

- [Wireshark Documentation](https://www.wireshark.org/docs/)
- [MITRE ATT&CK — Network Denial of Service (T1498)](https://attack.mitre.org/techniques/T1498/)
- [Cloudflare — DDoS Attack Types](https://www.cloudflare.com/learning/ddos/what-is-a-ddos-attack/)

---

*📺 Ikuti tutorialnya di [YouTube](https://youtu.be/E86twYGsINE?si=2kC3azBuhpivhH5_) | ⭐ Star repo ini jika membantu!*
