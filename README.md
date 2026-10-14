# 🌐 Administrasi Sistem Jaringan: VPN L2TP Site-to-Site MikroTik

Proyek ini adalah praktik **Uji Kompetensi Keahlian (UKK) Teknik Komputer dan Jaringan** (Kode KM.3066, Tahun Pelajaran 2025/2026). Dua router MikroTik di **Kantor Pusat Jakarta** dan **Kantor Cabang Surabaya** dihubungkan lewat tunnel **L2TP/IPsec** melalui internet, sehingga jaringan lokal kedua kantor bisa saling berkomunikasi.

> **Catatan:** angka `212` dan `112` pada alamat IP di bawah adalah contoh untuk **nomor absen 12**.
> Rumusnya: Jakarta = `192.(200 + absen)`, Surabaya = `192.(100 + absen)`.

---

## 📑 Daftar Isi
1. [Topologi](#-topologi)
2. [Alat dan Bahan](#-alat-dan-bahan)
3. [Persiapan Fisik](#-persiapan-fisik)
4. [Rencana Pengalamatan](#-rencana-pengalamatan)
5. [Konfigurasi Router Jakarta (L2TP Server)](#-konfigurasi-router-jakarta-l2tp-server)
6. [Konfigurasi Router Surabaya (L2TP Client)](#-konfigurasi-router-surabaya-l2tp-client)
7. [Pengujian](#-pengujian)
8. [Catatan Tambahan](#-catatan-tambahan)

---

## 🗺️ Topologi

<img width="673" height="443" alt="Topologi" src="https://github.com/user-attachments/assets/1586f1aa-4b81-49cc-af04-05520f2c2de1" />

---

## 🧰 Alat dan Bahan

| No | Item | Jumlah |
|----|------|--------|
| 1 | Koneksi internet dari ISP (minimal 1 Mbps) | 1 |
| 2 | Router MikroTik (minimal 4 port, mendukung routing static dan dynamic) | 2 |
| 3 | Wifi Routerboard (minimal 4 port) | 1 |
| 4 | Switch unmanaged (minimal 4 port) | 1 |
| 5 | PC Client / Laptop | 3 |
| 6 | Smartphone (Android/iOS) | 1 |
| 7 | Kabel UTP Cat 5 | 12 meter |
| 8 | Konektor RJ-45 Cat 5 | 10 pcs |
| 9 | Crimping tool dan cable tester | 1 set |
| 10 | Software virtualisasi dan OS server (open source) | 1 |
| 11 | Aplikasi kasir berbasis CMS (open source) | 1 |

---

## 🔌 Persiapan Fisik

1. Siapkan semua perangkat sesuai daftar di atas.
2. Pasang konektor RJ-45 pada kabel UTP, lalu uji kabel dengan cable tester.
3. Hubungkan kabel antara router, switch, dan PC client sesuai topologi.

---

## 📋 Rencana Pengalamatan

### Router Jakarta
| Keterangan | Interface | Alamat IP |
|-----------|-----------|-----------|
| Internet (ISP) | ether1 | DHCP dari ISP (contoh `172.18.20.252/24`) |
| Jaringan PC Kasir | ether2 | `192.212.10.1/24` |
| Jaringan Server POST | ether3 | `192.212.30.1/24` |
| Tunnel L2TP | - | Local `10.11.12.1` / Remote `10.11.12.2` |

### Router Surabaya
| Keterangan | Interface | Alamat IP |
|-----------|-----------|-----------|
| Internet (ISP) | ether1 | DHCP dari ISP (contoh `172.18.20.253/24`) |
| Jaringan PC Kasir | ether2 | `192.112.20.1/24` |

### PC Client
| Perangkat | IP Address | Gateway |
|-----------|-----------|---------|
| PC Kasir Jakarta | `192.212.10.2/24` | `192.212.10.1` |
| PC Kasir Surabaya | `192.112.20.2/24` | `192.112.20.1` |

DNS untuk kedua PC: DNS router sebagai Preferred dan `8.8.8.8` sebagai Alternate.

---

## 🏢 Konfigurasi Router Jakarta (L2TP Server)

Urutan pengerjaan: **Hostname → IP Address → DHCP Client → NAT → L2TP Server → Routing Static**

### 1. Hostname
**System → Identity**
- Identity: `Router Jakarta`

<img width="320" height="157" alt="Identity Jakarta" src="https://github.com/user-attachments/assets/c3d5fc42-2ab8-4946-b6d8-314ff909dc05" />

### 2. IP Address
**IP → Addresses → Add**

| Jaringan | Address | Interface |
|----------|---------|-----------|
| PC Kasir (`192.212.10.0/24`) | `192.212.10.1/24` | ether2 |
| Server POST (`192.212.30.0/24`) | `192.212.30.1/24` | ether3 |

<img width="342" height="162" alt="IP Address Jakarta" src="https://github.com/user-attachments/assets/52e1c0c2-c5c8-4f82-9c7e-d5192dba7379" />

### 3. DHCP Client
Ether1 sudah tersambung ke modem ISP, jadi alamat IP internet diambil otomatis.
**IP → DHCP Client → Add**
- Interface: `ether1`
- ✅ Use Peer DNS
- ✅ Use Peer NTP
- Add Default Route: `yes`

<img width="417" height="186" alt="DHCP Client Jakarta" src="https://github.com/user-attachments/assets/ebba0b9e-f446-4587-86a4-d09d31ad81cc" />

### 4. Firewall NAT
Router sudah online setelah mendapat IP dari ISP, tetapi client belum bisa mengakses internet sampai NAT dibuat.
**IP → Firewall → NAT → Add**
- Chain: `srcnat`
- Out. Interface: `ether1`
- Action: `masquerade`

<img width="727" height="276" alt="NAT Jakarta" src="https://github.com/user-attachments/assets/f271df2d-9c3c-467c-803e-f9fc53c83a58" />

### 5. L2TP Server
Router Jakarta berperan sebagai server VPN. Ada tiga langkah: profile, secret, lalu mengaktifkan service.

**a. PPP Profile** (**PPP → Profiles → Add**), untuk menentukan alamat tunnel:
- Name: `L2TP Jakarta <> Surabaya`
- Local Address: `10.11.12.1`
- Remote Address: `10.11.12.2`

<img width="433" height="162" alt="PPP Profile" src="https://github.com/user-attachments/assets/0b724098-37b4-43fe-94fe-fae017f5ac40" />

**b. PPP Secret** (**PPP → Secrets → Add**), supaya hanya router yang memegang kredensial ini yang bisa terhubung:
- Name: `Key-L2TP`
- Password: `P@ssw0rd`
- Service: `any`
- Profile: `L2TP Jakarta <> Surabaya`

<img width="507" height="191" alt="PPP Secret" src="https://github.com/user-attachments/assets/3a61d03b-f058-41a0-9b77-79400b0be88a" />

**c. Aktifkan L2TP Server** (**PPP → Interface → L2TP Server**). Secara default service ini nonaktif.
- ✅ Enabled
- Default Profile: `L2TP Jakarta <> Surabaya`
- Use IPsec: `yes`
- IPsec Secret: `P@ssw0rd`

<img width="452" height="353" alt="L2TP Server" src="https://github.com/user-attachments/assets/07635c1c-0319-426b-ab4d-41b804071545" />

### 6. Routing Static ke Surabaya
Gateway yang dipakai adalah IP tunnel milik Surabaya.
**IP → Routes → Add**
- Dst. Address: `192.112.20.0/24`
- Gateway: `10.11.12.2`

<img width="711" height="115" alt="Route Jakarta" src="https://github.com/user-attachments/assets/779edabc-547c-459a-a507-f4b7f04fac1a" />

✅ Konfigurasi Router Jakarta selesai.

---

## 🏬 Konfigurasi Router Surabaya (L2TP Client)

Urutan pengerjaan: **Hostname → IP Address → DHCP Client → NAT → L2TP Client → Routing Static**

### 1. Hostname
**System → Identity**
- Identity: `Router Surabaya`

<img width="313" height="152" alt="Identity Surabaya" src="https://github.com/user-attachments/assets/bfd2aaac-f525-4b2e-88e1-d17be9520332" />

### 2. IP Address
**IP → Addresses → Add**
- Address: `192.112.20.1/24`
- Network: `192.112.20.0`
- Interface: `ether2`

<img width="432" height="142" alt="IP Address Surabaya" src="https://github.com/user-attachments/assets/68e8cf50-92f1-48cf-8ca4-fa7c2e23eb56" />

### 3. DHCP Client
**IP → DHCP Client → Add**
- Interface: `ether1`
- ✅ Use Peer DNS
- ✅ Use Peer NTP
- Add Default Route: `yes`

<img width="420" height="187" alt="DHCP Client Surabaya" src="https://github.com/user-attachments/assets/9285e44a-3f13-4eb7-adc1-7424b1a44282" />

### 4. Firewall NAT
**IP → Firewall → NAT → Add**
- Chain: `srcnat`
- Out. Interface: `ether1`
- Action: `masquerade`

<img width="728" height="276" alt="NAT Surabaya" src="https://github.com/user-attachments/assets/c9675bdf-6845-4165-9452-60f30a167aeb" />

### 5. L2TP Client
Yang harus diketahui sebelum membuat client:
- IP publik Router Jakarta (alamat dari modem ISP)
- User dan password L2TP dari admin Kantor Pusat Jakarta

**PPP → Add → L2TP Client**
- Connect To: `172.18.20.252`
- User: `Key-L2TP`
- Password: `P@ssw0rd`
- ✅ Use IPsec
- IPsec Secret: `P@ssw0rd`

<img width="492" height="325" alt="L2TP Client" src="https://github.com/user-attachments/assets/bfddda8b-12b0-4e08-a915-42549931ef11" />

### 6. Routing Static ke Jakarta
Gateway yang dipakai adalah IP tunnel milik Jakarta.
**IP → Routes → Add**

| Dst. Address | Gateway |
|--------------|---------|
| `192.212.10.0/24` | `10.11.12.1` |
| `192.212.30.0/24` | `10.11.12.1` |

<img width="646" height="197" alt="Route Surabaya" src="https://github.com/user-attachments/assets/e692aeaa-3b91-4ba8-8265-33f0683b2199" />

✅ Konfigurasi Router Surabaya selesai.

---

## 🧪 Pengujian

Pengujian dilakukan dengan perintah `ping` lewat **CMD** dari PC client, lalu dilanjutkan dengan memeriksa status tunnel dan route di router.

### A. Sisi Jakarta (dari PC Kasir Jakarta)

<img width="352" height="336" alt="IP PC Jakarta" src="https://github.com/user-attachments/assets/83769df0-7257-45a8-b18b-8dddd85505c7" />

| Skenario | Tujuan |
|----------|--------|
| Ping gateway | `192.212.10.1` |
| Ping gateway server | `192.212.30.1` |
| Ping antar client | `192.112.20.1` (Surabaya) |
| Ping internet | `google.com` |

<img width="588" height="392" alt="Hasil Ping Jakarta" src="https://github.com/user-attachments/assets/d78f6270-8b06-4d92-b43e-f34d430e27a1" />

**Cek status L2TP** di **IP → Addresses** dan **PPP → Interface**. Jika berhasil, muncul interface L2TP baru dan alamat IP point-to-point dari tunnel.

<img width="540" height="308" alt="Status L2TP Jakarta" src="https://github.com/user-attachments/assets/d54d3e49-f35d-4b61-9558-67ef455ad018" />

**Cek routing** di **IP → Routes**. Route yang dibuat harus berflag `AS` (*Active Static*).

<img width="606" height="225" alt="Routes Jakarta" src="https://github.com/user-attachments/assets/ea017c2b-9f60-4e26-bc26-ff80795ed8e8" />

### B. Sisi Surabaya (dari PC Kasir Surabaya)

<img width="393" height="377" alt="IP PC Surabaya" src="https://github.com/user-attachments/assets/f2468403-b896-45ef-beea-8e7aad389787" />

| Skenario | Tujuan |
|----------|--------|
| Ping gateway | `192.112.20.1` |
| Ping gateway server | `192.212.30.1` |
| Ping antar client | `192.212.10.2` (PC Kasir Jakarta) |
| Ping internet | `google.com` |

<img width="592" height="395" alt="Hasil Ping Surabaya" src="https://github.com/user-attachments/assets/0a69f520-995a-435e-bcee-a9528917bfa3" />

**Cek status L2TP Client** di **IP → Addresses** dan **PPP → Interface**. Interface dan IP L2TP harus muncul otomatis seperti di Jakarta.

<img width="512" height="266" alt="Status L2TP Surabaya" src="https://github.com/user-attachments/assets/05c50ff9-600f-4875-8b65-7e5ac2962618" />

**Cek routing** di **IP → Routes**. Kedua route menuju Jakarta harus berflag `AS`.

<img width="587" height="217" alt="Routes Surabaya" src="https://github.com/user-attachments/assets/e052324a-f3a2-461c-8684-ffd7ade5e1ac" />

---

## ✅ Hasil yang Diharapkan

- Tunnel L2TP/IPsec antara Jakarta dan Surabaya berhasil terbentuk.
- Routing static aktif di kedua router (flag `AS`).
- PC Kasir kedua kantor saling terhubung dan bisa mengakses internet.
- Server POST di Jakarta bisa diakses dari PC Kasir Surabaya.

---

## 📝 Catatan Tambahan

Dokumentasi ini hanya membahas konfigurasi jaringan dan pengujiannya. Bagian berikut ada di soal UKK tetapi **belum dibahas** karena keterbatasan resource:

- Perangkat lunak virtualisasi dan sistem operasi server
- Web server dan database server
- Instalasi aplikasi kasir (CMS) pada Server POST
- Pengujian aplikasi, akses ke Server POST Jakarta, serta pengelolaan sistem cloud

---

<p align="center"><b>Selamat dan sukses! 🎉</b></p>
