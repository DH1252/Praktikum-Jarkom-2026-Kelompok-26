# Laporan Praktikum Modul 5: Vlan-Trunk-OSPF-MultiVendor

Praktikum ini membahas mengenai implementasi rancangan arsitektur jaringan skala besar (*Enterprise*) yang menghubungkan Kantor Pusat (HQ Jakarta) dan Kantor Cabang (Branch Surabaya) melewati jaringan *Internet Service Provider* (ISP) menggunakan teknologi *GRE Tunnel*, *dynamic routing OSPF over GRE*, redundansi *gateway VRRP*, serta distribusi alamat IP terpusat berbasis *ISC-DHCP Server*.

---

## 1. Topologi Jaringan
<img width="700" height="600" alt="image" src="https://github.com/user-attachments/assets/68b3bb7f-e67d-46a1-97dd-6c032b310c8e" />


---

## 2. Segmentasi Jaringan & Tabel Addressing

### 2.1 Jaringan Jakarta / HQ (VLAN & VRRP)
| VLAN | Nama VLAN | Network | Gateway Virtual | Keterangan |
| :--- | :--- | :--- | :--- | :--- |
| **10** | FINANCE | 192.168.10.0/24 | 192.168.10.1 | DHCP dari Ubuntu Server Jakarta (Master: Cisco, Backup: MikroTik) |
| **20** | IT | 192.168.20.0/24 | 192.168.20.1 | DHCP dari Ubuntu Server Jakarta (Master: MikroTik, Backup: Cisco) |
| **60** | SERVER-HQ | 192.168.60.0/24 | 192.168.60.1 | VLAN khusus server lokal Ubuntu Jakarta |

### 2.2 Tabel IP Address Perangkat

| Perangkat | Interface | IP Address | Gateway | Keterangan |
| :--- | :--- | :--- | :--- | :--- |
| **Cisco Router JKT** | Gi0/1.10 <br> Gi0/1.20 <br> Gi0/1.60 <br> Gi0/0 | 192.168.10.2/24 <br> 192.168.20.2/24 <br> 192.168.60.2/24 <br> 10.10.100.2/30 | - <br> - <br> - <br> 10.10.100.1 | IP fisik VLAN 10 <br> IP fisik VLAN 20 <br> IP fisik VLAN 60 <br> Link Transit ke FortiGate JKT |
| **MikroTik JKT** | vlan10-finance <br> vlan20-it <br> vlan60-server <br> ether1 | 192.168.10.3/24 <br> 192.168.20.3/24 <br> 192.168.60.3/24 <br> 10.10.101.2/30 | - <br> - <br> - <br> 10.10.101.1 | IP fisik VLAN 10 <br> IP fisik VLAN 20 <br> IP fisik VLAN 60 <br> Link Transit ke FortiGate JKT |
| **Ubuntu Server JKT**| eth0 | 192.168.60.10/24 | 192.168.60.1 | ISC-DHCP & Nginx Web Server |
| **FortiGate JKT** | port1 <br> port2 <br> port3 <br> GRE-JKT-SBY | 10.10.100.1/30 <br> 10.10.101.1/30 <br> 10.0.12.2/30 <br> 172.16.0.1/32 | - <br> - <br> 10.0.12.1 <br> - | Ke Cisco Router <br> Ke MikroTik Router <br> Link WAN ke ISP <br> IP Tunnel Interface |
| **MikroTik ISP** | ether1 <br> ether2 <br> ether3 | DHCP / Dinamis <br> 10.0.12.1/30 <br> 10.0.13.1/30 | Cloud NAT | Akses Internet luar <br> Link P2P ke FortiGate JKT <br> Link P2P ke FortiGate SBY |
| **FortiGate SBY** | port1 <br> port2 <br> GRE-SBY-JKT | 10.0.13.2/30 <br> 10.10.200.1/30 <br> 172.16.0.2/32 | 10.0.13.1 <br> - <br> - | Link WAN ke ISP <br> Link Transit ke MikroTik SBY <br> IP Tunnel Interface |
| **MikroTik SBY** | ether1 <br> vlan30-sales <br> vlan40-operations | 10.10.200.2/30 <br> 192.168.30.1/24 <br> 192.168.40.1/24 | 10.10.200.1 <br> - <br> - | Link Transit ke FortiGate SBY <br> Gateway VLAN 30 <br> Gateway VLAN 40 |

---

## 3. Langkah Pelaksanaan per Tugas Modul

### Tugas Modul 1 — Konfigurasi Cisco Switch Jakarta
* **Hal yang Dikonfigurasi:** Membuat VLAN 10, 20, dan 60; Mengatur port ke client VLAN 10 dan 20 sebagai access port; Mengatur port ke Ubuntu Server sebagai access VLAN 60; Mengatur link trunk menuju Cisco Router Jakarta dan MikroTik Router Jakarta yang membawa VLAN 10, 20, dan 60.
* **Hasil yang Diharapkan:** VLAN database terbentuk aktif; Client berada di segmen VLAN masing-masing; Jalur trunk menuju kedua router fungsional.
* **Bukti yang Dikumpulkan:**

#### Screenshot Topologi Jakarta


#### Screenshot `show vlan brief`


#### Screenshot `show interfaces trunk`


---

### Tugas Modul 2 — Konfigurasi Cisco Router Jakarta
* **Hal yang Dikonfigurasi:** Pembuatan sub-interface IEEE 802.1Q untuk VLAN 10, 20, dan 60 beserta IP fisiknya; Konfigurasi VRRP (Cisco sebagai Master untuk VLAN 10 & 60, Priority 120); Aktivasi DHCP Relay mengarah ke IP Ubuntu Server (`192.168.60.10`); Penambahan default rute transit ke FortiGate Jakarta.
* **Hasil yang Diharapkan:** Sub-interface berstatus Up/Up; Router mengambil peran VRRP Master/Backup sesuai rancangan; Mampu meneruskan paket DHCP request.
* **Bukti yang Dikumpulkan:**

#### Screenshot `show ip interface brief`


#### Screenshot `show vrrp brief`


#### Screenshot Konfigurasi Subinterface & DHCP Relay


#### Screenshot Ping dari Cisco Router ke FortiGate Jakarta (`10.10.100.1`)


---

### Tugas Modul 3 — Konfigurasi MikroTik Router Jakarta
* **Hal yang Dikonfigurasi:** Pembuatan VLAN interface pada ether2 menuju switch; Pemberian IP fisik; Konfigurasi VRRP (MikroTik sebagai Master untuk VLAN 20, Priority 120; dan Backup untuk VLAN 10 & 60, Priority 90); Pembuatan DHCP Relay; Pengaturan rute statis default via `10.10.101.1`.
* **Hasil yang Diharapkan:** VLAN interface terbentuk; Sinkronisasi VRRP bersama Cisco berjalan seimbang; Transit rute ke FortiGate aktif.
* **Bukti yang Dikumpulkan:**

#### Screenshot `/ip address print`


#### Screenshot `/interface vrrp print`


#### Screenshot `/ip dhcp-relay print`


#### Screenshot `/ip route print`


#### Screenshot Ping dari MikroTik ke FortiGate Jakarta (`10.10.101.1`)


---

### Tugas Modul 4 — Konfigurasi Ubuntu Server Jakarta
* **Hal yang Dikonfigurasi:** Pengaturan IP statis ens3/eth0 pada VLAN 60; Penetapan gateway ke IP virtual VRRP `192.168.60.1`; Instalasi paket `isc-dhcp-server` dan `nginx`; Konfigurasi pool scope subnet DHCP untuk VLAN 10 dan VLAN 20 dengan option routers mengarah ke virtual IP VRRP.
* **Hasil yang Diharapkan:** Layanan DHCP Server berjalan aktif tanpa error; Nginx web server dapat diakses; Distribusi IP ke client eksternal VLAN sukses.
* **Bukti yang Dikumpulkan:**

#### Screenshot `ip a`


#### Screenshot `ip route`


#### Screenshot Isi File `/etc/dhcp/dhcpd.conf`


#### Screenshot Ping dari Ubuntu ke Internet (`8.8.8.8`)


---

### Tugas Modul 5 — Konfigurasi FortiGate Jakarta
* **Hal yang Dikonfigurasi:** IP static port1 (Cisco), port2 (MikroTik), port3 (ISP); Default rute ke ISP; Rute statis ke network internal Jakarta (`192.168.0.0/16`); Firewall policy inbound/outbound dengan parameter NAT aktif; Inisialisasi interface GRE Tunnel dan OSPF area 0 over GRE disertai aksi redistribute static.
* **Hasil yang Diharapkan:** Terkoneksi ke ISP; Dapat menjangkau internet luar; Terbentuknya rute inter-site dinamis OSPF.
* **Bukti yang Dikumpulkan:**

#### Screenshot `get system interface physical`


#### Screenshot `get router info routing-table all`


#### Screenshot Firewall Policy (NAT Enabled)


#### Screenshot Ping dari FortiGate Jakarta ke Internet (`8.8.8.8`)


#### Screenshot Ping ke IP Tunnel Surabaya (`172.16.0.2`)


#### Screenshot `get router info ospf neighbor`


#### Screenshot `get router info routing-table ospf`


---

### Tugas Modul 6 — Konfigurasi MikroTik ISP
* **Hal yang Dikonfigurasi:** IP address pada ether2 (ke JKT) dan ether3 (ke SBY); Aktivasi DHCP Client pada ether1 kearah Cloud PNETLab; Default route kearah cloud; Konfigurasi Firewall NAT masquerade luar pada out-interface ether1.
* **Hasil yang Diharapkan:** Menjadi jembatan transit WAN; Menyediakan akses NAT internet luar; Tidak menjalankan protokol OSPF internal enterprise.
* **Bukti yang Dikumpulkan:**

#### Screenshot `/ip address print`


#### Screenshot `/ip route print`


#### Screenshot `/ip firewall nat print`


#### Screenshot Ping dari ISP ke Internet (`8.8.8.8`)


#### Screenshot Ping Antar-WAN FortiGate (JKT $\leftrightarrow$ SBY) via ISP


---

### Tugas Modul 7 — Konfigurasi Switch dan MikroTik Surabaya
* **Hal yang Dikonfigurasi:** Pembuatan VLAN 30 dan 40 di Switch Surabaya beserta port access/trunk; Pembuatan VLAN interface di MikroTik Surabaya; Konfigurasi IP gateway; Aktivasi DHCP Server lokal untuk VLAN 30 Sales; Setup IP static manual untuk VLAN 40 Operations; Default route ke FortiGate Surabaya.
* **Hasil yang Diharapkan:** Segmen cabang Surabaya terbentuk; Client VLAN 30 mendapat IP dinamis; Seluruh client Surabaya terhubung ke gateway transit.
* **Bukti yang Dikumpulkan:**

#### Screenshot `show vlan brief` (Switch Surabaya)


#### Screenshot `show interfaces trunk` (Switch Surabaya)


#### Screenshot `/ip address print` (MikroTik Surabaya)


#### Screenshot `/ip dhcp-server print` & `/ip pool print`


#### Screenshot `/ip route print` (MikroTik Surabaya)


#### Screenshot Client VLAN 30 Mendapat IP DHCP (`show ip`)


#### Screenshot Ping Client Surabaya ke Internet (`8.8.8.8`)


---

### Tugas Modul 8 — Konfigurasi FortiGate Surabaya
* **Hal yang Dikonfigurasi:** IP static port1 (ke ISP), port2 (ke MikroTik SBY); Default route ke ISP; Rute statis menuju network internal Surabaya via MikroTik SBY (`192.168.30.0/24` & `192.168.40.0/24`); Firewall policy NAT internet; Inisialisasi interface GRE Tunnel dan OSPF over GRE.
* **Hasil yang Diharapkan:** FortiGate SBY terhubung ke internet luar; Terbentuknya rute inter-site dinamis OSPF kearah Jakarta.
* **Bukti yang Dikumpulkan:**

#### Screenshot `get system interface physical`


#### Screenshot `get router info routing-table all`


#### Screenshot Firewall Policy Surabaya


#### Screenshot Ping dari FortiGate Surabaya ke Internet (`8.8.8.8`)


#### Screenshot Ping ke IP Tunnel Jakarta (`172.16.0.1`)


#### Screenshot `get router info ospf neighbor`


#### Screenshot `get router info routing-table ospf`


---

### Tugas Modul 9 — Konfigurasi GRE Tunnel dan OSPF over GRE
* **Hal yang Dikonfigurasi:** Pembuatan parameterisasi ujung terowongan virtual (*GRE endpoint*) timbal balik lintas area; Pemasangan IP terowongan; Aktivasi pertukaran database dynamic routing backbone Area 0 secara penuh.
* **Hasil yang Diharapkan:** Koneksi logika antar-headquarter terjalin; Distribusi rute internal tersebar otomatis; Kedua belah pihak saling mengenali segmen IP privat lawan.
* **Bukti yang Dikumpulkan:**

#### Screenshot Ping WAN Antar-FortiGate


#### Screenshot Ping Tunnel Antar-FortiGate


#### Screenshot `get router info ospf neighbor` & `routing-table ospf` (Kedua Sisi)


#### Screenshot Ping Silang Client Jakarta $\leftrightarrow$ Client Surabaya


---

### Tugas Modul 10 — Pengujian Akhir
* **Hal yang Diuji:** Validasi penerimaan IP DHCP terpusat (Jakarta) dan lokal (Surabaya); Aksesibilitas internet publik dari kedua kantor; Tes jangkauan koneksi antar-site secara end-to-end; Akses web server Nginx Ubuntu dari komputer cabang Surabaya.
* **Hasil yang Diharapkan:** Seluruh uji fungsi sukses; Redundansi VRRP bekerja normal saat failover; Web server Jakarta termuat lancar dari Surabaya.
* **Bukti yang Dikumpulkan:**

#### Screenshot IP DHCP Client Jakarta (VLAN 10)


#### Screenshot IP DHCP Client Surabaya (VLAN 30)


#### Screenshot Ping Internet (`8.8.8.8`) dari Jakarta


#### Screenshot Ping Internet (`8.8.8.8`) dari Surabaya


#### Screenshot Ping Antar-Site (Jakarta $\leftrightarrow$ Surabaya)


#### Screenshot Akses Halaman Web Server Jakarta (`192.168.60.10`) dari Surabaya


---

## 4. Analisis Jalur Traffic Jarsby (Jakarta ke Surabaya)

Aliran jalannya paket data dari Client Jakarta saat ingin menuju ke Client Surabaya dijabarkan secara runut melewati titik lompatan (*hop*) berikut:

1.  **Inisiasi Client:** Client Jakarta mengirimkan paket data menuju alamat IP tujuan Surabaya (`192.168.30.0/24` atau `192.168.40.0/24`). Paket pertama kali dilemparkan ke arah alamat IP Virtual VRRP (`192.168.10.1` atau `192.168.20.1`) sebagai *default gateway*-nya.
2.  **Gateway Internal (VRRP Node):** Perangkat yang bertindak sebagai *VRRP Master* (Cisco Router atau MikroTik Router Jakarta) menerima paket tersebut, membaca tabel routing-nya, lalu meneruskan paket ke arah interface internal FortiGate Jakarta via jalur transit (`10.10.100.1` atau `10.10.101.1`).
3.  **Edge Firewall (FortiGate Jakarta):** FortiGate Jakarta memeriksa aturan keamanan (*security policy*). Karena rute tujuan eksternal tersebut dipelajari via OSPF melalui interface virtual, maka FortiGate melakukan enkapsulasi paket data ke dalam protokol GRE, lalu membungkusnya dengan IP WAN luar.
4.  **Transport Provider (ISP):** Paket terenkapsulasi dikirim keluar melalui link WAN port 3 menuju MikroTik ISP (`10.0.12.1`). MikroTik ISP murni hanya melakukan *routing* paket publik menuju ke IP WAN FortiGate Surabaya (`10.0.13.2`) tanpa pernah tahu isi paket data privat di dalamnya.
5.  **Decapsulation Endpoint (FortiGate Surabaya):** FortiGate Surabaya menerima paket pada port 1, mengidentifikasi bahwa paket tersebut ditujukan untuk interface GRE miliknya, lalu melakukan pembongkaran enkapsulasi (*decapsulation*) untuk mengambil paket data asli.
6.  **Distribusi Lokal Cabang:** Paket asli kemudian diteruskan ke arah MikroTik Surabaya melalui link transit internal (`10.10.200.2`), hingga akhirnya didistribusikan oleh Switch Surabaya langsung menuju ke PC *end-device* tujuan di Kantor Cabang.

---

## 5. Kesimpulan

1.  **Redundansi Tingkat Tinggi Berhasil Diuji:** Implementasi VRRP menggunakan kombinasi *multi-vendor* antara Cisco Router dan MikroTik Router sukses menyediakan fungsi dual-gateway yang andal untuk mengantisipasi kegagalan sistem lokal (*Single Point of Failure*).
2.  **Koneksi Virtual Lintas Wilayah Aman:** Pemanfaatan *GRE Tunnel* yang dipadukan dengan *OSPF over GRE* terbukti efektif menghubungkan dua klaster jaringan kantor terpisah melewati infrastruktur publik/ISP secara dinamis tanpa perlu melakukan perubahan rute manual.
3.  **Pengelolaan IP Terpusat Efisien:** Penggunaan layanan *ISC-DHCP Server* di Ubuntu Server yang dikombinasikan dengan fitur *DHCP Relay* pada router mampu memusatkan manajemen distribusi IP Address jaringan internal enterprise secara rapi dan otomatis.
