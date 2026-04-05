# 🌊 DDoS Attack Detection — Analisis dengan Splunk SIEM

> **Seri Lab:** Blue Team SOC Lab  
> **Platform:** [TryHackMe](https://tryhackme.com)  
> **YouTube:** [▶️ Tonton Tutorial di YouTube](https://youtu.be/yWcJ6MNcjhk?si=JZ4PbwUg9oSgE70E)  
> **Kategori:** `SIEM Analysis, Threat Detection`

---

## 📋 Deskripsi

Dokumentasi ini menjelaskan proses deteksi serangan **Distributed Denial of Service (DDoS)** menggunakan **Splunk** berdasarkan analisis log web server (Apache/Nginx). Berbeda dengan analisis manual menggunakan Wireshark, pendekatan SIEM memungkinkan deteksi yang lebih cepat dan dapat diterapkan pada skala traffic yang jauh lebih besar.

**Pendekatan ini digunakan untuk:**
- Mengidentifikasi lonjakan traffic secara real-time
- Menemukan IP attacker dari ribuan log sekaligus
- Menganalisis endpoint yang menjadi target serangan
- Menilai dampak serangan terhadap server

---

## 🎬 Video Tutorial

[![DDoS Detection - SIEM Analysis with Splunk](https://img.youtube.com/vi/yWcJ6MNcjhk/maxresdefault.jpg)](https://youtu.be/yWcJ6MNcjhk?si=JZ4PbwUg9oSgE70E)

> 📺 **[Tonton di YouTube → DDoS Attack Detection dengan Splunk SIEM](https://youtu.be/yWcJ6MNcjhk?si=JZ4PbwUg9oSgE70E)**

---

## 🎯 Tujuan Investigasi

- Mendeteksi aktivitas DDoS dari log web server
- Mengidentifikasi IP yang mencurigakan
- Menentukan endpoint yang diserang
- Menganalisis dampak serangan ke sistem

---

## 🛠️ Tools yang Digunakan

| Tool | Fungsi |
|------|--------|
| **Splunk** | Analisis dan korelasi log secara terpusat |
| **Web Server Logs** | Sumber data (Apache access.log / Nginx log) |

---

## 🚨 Indikator Serangan DDoS

| Indikator | Penjelasan |
|-----------|------------|
| Lonjakan request dalam waktu singkat | Volume traffic naik drastis dalam hitungan detik |
| Banyak request ke endpoint yang sama | Satu URI menjadi target masif |
| Banyak IP menyerang secara bersamaan | Ciri khas DDoS — bukan DoS dari satu sumber |
| Error 503 Service Unavailable | Server sudah overload akibat serangan |

---

## 🔍 Langkah-Langkah Analisis

### Langkah 1 — Pastikan Data Sudah Masuk ke Splunk

Verifikasi log sudah ter-index dengan query dasar:

```splunk
index="main"
```

> 💡 Log bisa berasal dari Apache `access.log`, Nginx log, atau dataset yang disediakan lab TryHackMe.

---

### Langkah 2 — Deteksi Lonjakan Traffic (Traffic Spike)

Visualisasikan jumlah request per detik untuk menemukan lonjakan:

```splunk
index="main"
| timechart span=1s count
```

**Cara membaca hasilnya:**

```
Traffic normal  → grafik stabil / naik perlahan
Traffic DDoS    → spike vertikal tajam dalam hitungan detik
```

> 💡 Gunakan tampilan **Visualization → Column Chart** agar lonjakan lebih mudah terlihat.

---

### Langkah 3 — Identifikasi IP dengan Request Terbanyak

```splunk
index="main"
| stats count by clientip
| sort -count
```

**Yang dicari:**
- IP dengan jumlah request jauh di atas rata-rata
- Pada serangan DDoS, akan ada **banyak IP berbeda** dengan request tinggi secara bersamaan

---

### Langkah 4 — Analisis Endpoint yang Diserang

```splunk
index="main"
| top uri
```

**Endpoint yang sering menjadi target:**

| Endpoint | Jenis Serangan |
|----------|----------------|
| `/login` | Login flooding / credential stuffing |
| `/api` | API abuse |
| `/index.php` | Homepage flooding |
| `/search` | Search query flooding |

> 💡 Endpoint dengan jumlah request tertinggi adalah **target utama** serangan.

---

### Langkah 5 — Analisis User-Agent (Identifikasi Tool Attacker)

```splunk
index="main"
| top useragent
```

**User-Agent yang mencurigakan:**

| User-Agent | Keterangan |
|------------|------------|
| `curl/x.x.x` | Tool command line — bukan browser |
| `python-requests/x.x` | Script Python otomatis |
| User-Agent identik semua | Kemungkinan spoofed atau bot |
| Kosong / tidak ada | Sangat mencurigakan |

---

### Langkah 6 — Analisis Error (Dampak ke Server)

```splunk
index="main" status=503
```

> **HTTP 503 Service Unavailable** adalah tanda bahwa server sudah **overload** dan tidak mampu melayani request. Semakin banyak 503 yang muncul, semakin besar dampak serangan terhadap layanan.

---

### Langkah 7 — Cek User Internal yang Terdampak

```splunk
index="main" status=503 clientip=10*
```

> 💡 Query ini memfilter error 503 yang dialami oleh IP internal (10.x.x.x). Jika user internal juga terdampak, artinya serangan sudah mengganggu operasional jaringan secara keseluruhan — bukan hanya pengguna eksternal.

---

### Langkah 8 — Korelasi Temuan (Konfirmasi DDoS)

Gabungkan semua insight dari langkah sebelumnya:

```splunk
index="main"
| stats count by clientip, uri, status
| sort -count
```

**Pola DDoS yang terkonfirmasi:**

```
✅ Banyak IP berbeda dengan request tinggi
✅ Satu endpoint menjadi target dominan
✅ Lonjakan traffic dalam waktu sangat singkat
✅ Server menghasilkan banyak error 503
```

**Contoh pola nyata:**

```
5.000 request dalam 1 menit ke /login dari 200+ IP berbeda
→ Kesimpulan: DDoS Attack (Application Layer — HTTP Flood)
```

---

## 📊 Bagian 9 — Dashboard Monitoring Real-Time

Buat dashboard di Splunk untuk monitoring berkelanjutan:

### Panel yang Disarankan

```splunk
# Panel 1 — Traffic per detik
index="main"
| timechart span=1s count

# Panel 2 — Top IP Attacker
index="main"
| stats count by clientip
| sort -count
| head 10

# Panel 3 — Top Endpoint yang Diserang
index="main"
| top limit=10 uri

# Panel 4 — Error Rate (503)
index="main" status=503
| timechart span=1m count

# Panel 5 — User-Agent Mencurigakan
index="main"
| top limit=10 useragent
```

### Cara Membuat Dashboard:

```
1. Jalankan query di Search & Reporting
2. Klik Save As → Dashboard Panel
3. Buat dashboard baru: "DDoS Monitoring"
4. Tambahkan semua panel di atas
5. Set auto-refresh sesuai kebutuhan
```

---

## 🚨 Setup Alert Otomatis

Buat alert untuk mendeteksi DDoS secara otomatis:

```splunk
index="main"
| timechart span=1m count
| where count > 1000
```

**Konfigurasi Alert:**

```
Title       : DDoS Traffic Spike Alert
Alert Type  : Real-time
Trigger     : Number of Results > 0
Actions     : Log event / Send email
```

---

## ✅ Ringkasan Temuan

| Aspek | Temuan |
|-------|--------|
| **Pola Traffic** | Lonjakan signifikan dalam waktu sangat singkat |
| **Target** | Endpoint spesifik menerima request masif |
| **Sumber** | Banyak IP berbeda — terdistribusi |
| **Dampak** | Server menghasilkan banyak error 503 |
| **Kategori** | DDoS Attack — Application Layer (HTTP Flood) |

---

## 🛡️ Rekomendasi Mitigasi

### Level Aplikasi

| Mitigasi | Penjelasan |
|----------|------------|
| **Rate Limiting** | Batasi jumlah request per IP per detik |
| **CAPTCHA** | Tambahkan di endpoint sensitif seperti `/login` |

### Level Jaringan

| Mitigasi | Penjelasan |
|----------|------------|
| **CDN (Cloudflare)** | Distribusi dan absorpsi traffic serangan |
| **Web Application Firewall (WAF)** | Filter request berbahaya sebelum ke server |

### Monitoring Berkelanjutan

| Mitigasi | Penjelasan |
|----------|------------|
| **Splunk Alert** | Notifikasi otomatis saat traffic spike terdeteksi |
| **Real-time Dashboard** | Pantau kondisi server secara terus-menerus |

---

## 📌 Perbandingan: Manual (Wireshark) vs SIEM (Splunk)

| Aspek | Wireshark (Manual) | Splunk (SIEM) |
|-------|-------------------|----------------|
| **Kecepatan** | Lambat untuk data besar | Cepat — ratusan ribu log dalam detik |
| **Skala** | Cocok untuk PCAP kecil | Cocok untuk skala enterprise |
| **Detail** | Sangat detail (packet level) | Summary dan pola (log level) |
| **Otomasi** | Tidak ada | Bisa dibuat alert & dashboard |
| **Kegunaan** | Investigasi forensik mendalam | Deteksi real-time & monitoring |
| **User** | Analis forensik | SOC Analyst L1 & L2 |

> 💡 **Kombinasi terbaik:** Splunk untuk deteksi awal real-time → Wireshark untuk investigasi forensik mendalam setelah insiden teridentifikasi.

---

## 📚 Referensi

- [Splunk Search Reference](https://docs.splunk.com/Documentation/Splunk/latest/SearchReference)
- [MITRE ATT&CK — Network Denial of Service (T1498)](https://attack.mitre.org/techniques/T1498/)
- [Cloudflare — Understanding DDoS Attacks](https://www.cloudflare.com/learning/ddos/what-is-a-ddos-attack/)
- [OWASP — Denial of Service](https://owasp.org/www-community/attacks/Denial_of_Service)

---

*📺 Ikuti tutorialnya di [YouTube](https://youtu.be/yWcJ6MNcjhk?si=JZ4PbwUg9oSgE70E) | ⭐ Star repo ini jika membantu!*
