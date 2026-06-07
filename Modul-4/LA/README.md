# Laporan Praktikum Modul 4: DMZ & Firewall

Praktikum ini membahas mengenai implementasi segmentasi jaringan menggunakan Firewall FortiGate, Router Cisco, dan Router MikroTik ISP untuk mengamankan jaringan internal (LAN) serta menyediakan akses terbatas ke server publik (DMZ).

---

## 1. Topologi Jaringan
<img width="600" height="500" alt="image" src="https://github.com/user-attachments/assets/3799c42e-8ed0-4e9b-8e3b-74c3ca9ad77c" />

---

## 2. Segmentasi Jaringan & Tabel IP Address

### Segmentasi Jaringan
| Segment | Network | Keterangan |
| :--- | :--- | :--- |
| Jaringan Lab / Internet | DHCP | Sumber koneksi luar dari lab |
| ISP ke FortiGate | 10.10.10.0/30 | Link interkoneksi MikroTik ISP dan FortiGate |
| Client WAN | 172.16.100.0/24 | Jaringan client dari sisi luar / internet simulasi |
| FortiGate ke Cisco | 10.20.20.0/30 | Link interkoneksi FortiGate dan Cisco Router |
| LAN | 192.168.10.0/24 | Jaringan internal client |
| DMZ | 192.168.20.0/24 | Jaringan server publik DMZ |

### Tabel IP Address Perangkat
| Perangkat | Interface | IP Address | Gateway | Keterangan |
| :--- | :--- | :--- | :--- | :--- |
| **MikroTik ISP** | ether1 <br> ether2 <br> ether3 | DHCP Client <br> 10.10.10.1/30 <br> 172.16.100.1/24 | - <br> - <br> - | Terhubung ke Cloud <br> Terhubung ke FortiGate port1 <br> Gateway Client WAN |
| **FortiGate** | port1 <br> port2 <br> port3 | 10.10.10.2/30 <br> 10.20.20.1/30 <br> 192.168.20.1/24 | 10.10.10.1 <br> - <br> - | Interface WAN <br> Interface INSIDE ke Cisco <br> Interface DMZ |
| **Cisco Router** | G0/0 <br> G0/1 | 10.20.20.2/30 <br> 192.168.10.1/24 | - <br> - | Terhubung ke FortiGate port2 <br> Gateway LAN |
| **Client LAN** | eth0 | 192.168.10.10/24 | 192.168.10.1 | Tinycore Linux (Client Internal) |
| **Client WAN** | eth0 | 172.16.100.10/24 | 172.16.100.1 | Tinycore Linux (Client External) |
| **Ubuntu DMZ** | eth0 / ens3 | 192.168.20.10/24 | 192.168.20.1 | Web Server Nginx |

---

## 3. Konfigurasi Perangkat

### 3.1 MikroTik ISP
Konfigurasi meliputi IP Address, DHCP Client pada ether1, NAT Masquerade ke arah internet, dan Static Route menuju jaringan LAN dan DMZ via FortiGate.


### 3.2 FortiGate Firewall
Konfigurasi interface, Default Route, Static Route ke arah LAN, Address Objects, Security Policy, dan Virtual IP (Port Forwarding HTTP).


### 3.3 Cisco Router
Aktivasi interface dan pembuatan default route menuju FortiGate.


### 3.4 Ubuntu Server DMZ
Pengaturan IP Statis dan instalasi serta konfigurasi web server Nginx.


## 4. Hasil Pengujian dan Analisis
### 4.1 Pengujian Client LAN ke Gateway Cisco
Target: Sukses Ping ke 192.168.10.1.

Bukti: [Masukkan Gambar Screenshot]

Analisis: Client LAN berhasil menjangkau interface internal router sebagai default gateway-nya.

### 4.2 Pengujian Client LAN ke FortiGate
Target: Sukses Ping ke 10.20.20.1.

Bukti: [Masukkan Gambar Screenshot]

Analisis: Jalur routing dari Cisco Router ke arah FortiGate berjalan dengan baik.

### 4.3 Pengujian Client LAN ke DMZ
Target: Sukses Ping ke IP asli DMZ 192.168.20.10.

Bukti: [Masukkan Gambar Screenshot]

Analisis: Diperbolehkan karena policy LAN_to_DMZ pada FortiGate mengizinkan paket melewati firewall.

### 4.4 Pengujian Client LAN Akses IP DMZ via HTTP
Target: Berhasil membuka web server http://192.168.20.10.

Bukti: [Masukkan Gambar Screenshot]

Analisis: Protokol HTTP (TCP port 80) diizinkan dari area LAN menuju DMZ.

### 4.5 Pengujian Client WAN Ping ke ISP MikroTik
Target: Sukses Ping ke 172.16.100.1.

Bukti: [Masukkan Gambar Screenshot]

Analisis: Client WAN terhubung langsung ke interface ether3 MikroTik ISP.

### 4.6 Pengujian Client WAN Ping ke FortiGate
Target: Sukses Ping ke 10.10.10.2.

Bukti: [Masukkan Gambar Screenshot]

Analisis: MikroTik meneruskan paket ping karena memiliki rute langsung (Directly Connected) ke segmen WAN FortiGate.

### 4.7 Pengujian Client WAN Akses HTTP melalui IP Publik/VIP
Target: Berhasil mengakses http://10.10.10.2 dan memunculkan custom page Nginx.

Bukti: [Masukkan Gambar Screenshot]

Analisis: Virtual IP (VIP) Port Forwarding di FortiGate berhasil meneruskan request HTTP eksternal ke IP internal server DMZ.

### 4.8 Pengujian Client WAN Ping ke Client LAN
Target: REJECTED / RTO (Tidak Bisa Ping ke 192.168.10.10).

Bukti: [Masukkan Gambar Screenshot]

Analisis: Firewall FortiGate secara default memblokir koneksi dari luar (WAN) langsung ke dalam jaringan lokal (LAN) demi aspek keamanan.

### 4.9 Pengujian Client WAN Ping ke IP Asli DMZ
Target: REJECTED / RTO (Tidak Bisa Ping ke 192.168.20.10).

Bukti: [Masukkan Gambar Screenshot]

Analisis: Jaringan luar dilarang mengetahui atau berinteraksi langsung dengan IP asli DMZ kecuali melewati mekanisme VIP / Port Forwarding yang telah ditentukan.

### 4.10 Pengujian Server DMZ Ping ke LAN
Target: REJECTED / RTO (Tidak Bisa Ping ke 192.168.10.10).

Bukti: [Masukkan Gambar Screenshot]

Analisis: Zona DMZ adalah zona semi-trusted, sehingga inisiasi koneksi baru dari DMZ menuju LAN diblokir oleh kebijakan keamanan firewall untuk mencegah peretasan meluas jika server web berkompromi.

## Kesimpulan
Segmentasi Jaringan Berhasil: Pemisahan zona menjadi WAN, LAN, dan DMZ menggunakan FortiGate berhasil membatasi hak akses antar segmen sesuai kebijakan keamanan (Security Policy).

Fungsi VIP (Virtual IP): Mekanisme Port Forwarding terbukti efektif mempublikasikan layanan web internal (DMZ) ke jaringan luar tanpa mengekspos alamat IP lokal server yang sebenarnya.

Prinsip DMZ Keamanan Tinggi: Terbukti bahwa zona LAN dapat mengakses DMZ dan WAN, namun area luar (WAN) dan area semi-aman (DMZ) tidak diberikan izin menginisiasi koneksi langsung ke zona lokal (LAN).
