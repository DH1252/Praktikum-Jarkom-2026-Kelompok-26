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
<img width="200" height="400" alt="image" src="https://github.com/user-attachments/assets/1ffb03eb-48e2-4904-9fae-12e64a5f62c0" />


#### Screenshot `show vlan brief`
<img width="1108" height="629" alt="image" src="https://github.com/user-attachments/assets/5f22ec96-9223-480a-9198-043effd5343b" />


#### Screenshot `show interfaces trunk`
<img width="1128" height="672" alt="image" src="https://github.com/user-attachments/assets/ca52676b-2a34-4c2e-949a-e154ccd4b758" />


---

### Tugas Modul 2 — Konfigurasi Cisco Router Jakarta
* **Hal yang Dikonfigurasi:** Pembuatan sub-interface IEEE 802.1Q untuk VLAN 10, 20, dan 60 beserta IP fisiknya; Konfigurasi VRRP (Cisco sebagai Master untuk VLAN 10 & 60, Priority 120); Aktivasi DHCP Relay mengarah ke IP Ubuntu Server (`192.168.60.10`); Penambahan default rute transit ke FortiGate Jakarta.
* **Hasil yang Diharapkan:** Sub-interface berstatus Up/Up; Router mengambil peran VRRP Master/Backup sesuai rancangan; Mampu meneruskan paket DHCP request.
* **Bukti yang Dikumpulkan:**

#### Screenshot `show ip interface brief`
<img width="1134" height="739" alt="image" src="https://github.com/user-attachments/assets/de4c78f2-0d21-4762-892b-3256214fcaa4" />


#### Screenshot `show vrrp brief`
<img width="1125" height="649" alt="image" src="https://github.com/user-attachments/assets/5722ee8a-a77f-45c2-a8b4-4cbcafcafeb9" />


#### Screenshot Ping dari Cisco Router ke FortiGate Jakarta (`10.10.100.1`)
<img width="1156" height="662" alt="image" src="https://github.com/user-attachments/assets/56f628c2-8b9e-4894-a6dc-147ee180ff36" />


---

### Tugas Modul 3 — Konfigurasi MikroTik Router Jakarta
* **Hal yang Dikonfigurasi:** Pembuatan VLAN interface pada ether2 menuju switch; Pemberian IP fisik; Konfigurasi VRRP (MikroTik sebagai Master untuk VLAN 20, Priority 120; dan Backup untuk VLAN 10 & 60, Priority 90); Pembuatan DHCP Relay; Pengaturan rute statis default via `10.10.101.1`.
* **Hasil yang Diharapkan:** VLAN interface terbentuk; Sinkronisasi VRRP bersama Cisco berjalan seimbang; Transit rute ke FortiGate aktif.
* **Bukti yang Dikumpulkan:**

#### Screenshot `/ip address print`
<img width="1096" height="642" alt="image" src="https://github.com/user-attachments/assets/ba58e045-7728-40a5-9485-18c6e33e2d58" />


#### Screenshot `/interface vrrp print`
<img width="1134" height="654" alt="image" src="https://github.com/user-attachments/assets/e6d9864e-331e-466b-a72b-905fb2fab854" />


#### Screenshot `/ip dhcp-relay print`
<img width="1152" height="677" alt="image" src="https://github.com/user-attachments/assets/5e6f6876-5e5a-49ac-98a0-1efbfa80b03c" />


#### Screenshot `/ip route print`
<img width="1137" height="668" alt="image" src="https://github.com/user-attachments/assets/714e7fb8-2568-42ae-8556-4edaf67bb654" />


#### Screenshot Ping dari MikroTik ke FortiGate Jakarta (`10.10.101.1`)
<img width="1112" height="662" alt="image" src="https://github.com/user-attachments/assets/94c78ce3-7590-4e6e-84cc-c04c1eb73b90" />


---

### Tugas Modul 4 — Konfigurasi Ubuntu Server Jakarta
* **Hal yang Dikonfigurasi:** Pengaturan IP statis ens3/eth0 pada VLAN 60; Penetapan gateway ke IP virtual VRRP `192.168.60.1`; Instalasi paket `isc-dhcp-server` dan `nginx`; Konfigurasi pool scope subnet DHCP untuk VLAN 10 dan VLAN 20 dengan option routers mengarah ke virtual IP VRRP.
* **Hasil yang Diharapkan:** Layanan DHCP Server berjalan aktif tanpa error; Nginx web server dapat diakses; Distribusi IP ke client eksternal VLAN sukses.
* **Bukti yang Dikumpulkan:**

#### Screenshot `ip a`
<img width="1275" height="554" alt="image" src="https://github.com/user-attachments/assets/38dc9ef8-86ca-40b1-8856-d68eeff2820c" />


#### Screenshot `ip route`
<img width="1275" height="554" alt="image" src="https://github.com/user-attachments/assets/5dd28c79-6d17-49a0-83a5-6f25a6f6c07b" />


#### Screenshot Isi File `/etc/dhcp/dhcpd.conf`
<img width="1285" height="1249" alt="image" src="https://github.com/user-attachments/assets/dd5af369-9ab5-4f3f-8df1-b3858a55adb0" />


#### Screenshot Ping dari Ubuntu ke Internet (`8.8.8.8`)
<img width="1288" height="898" alt="image" src="https://github.com/user-attachments/assets/a5c0647f-98e5-40d9-82f3-c4f2fb92e48d" />


---

### Tugas Modul 5 — Konfigurasi FortiGate Jakarta
* **Hal yang Dikonfigurasi:** IP static port1 (Cisco), port2 (MikroTik), port3 (ISP); Default rute ke ISP; Rute statis ke network internal Jakarta (`192.168.0.0/16`); Firewall policy inbound/outbound dengan parameter NAT aktif; Inisialisasi interface GRE Tunnel dan OSPF area 0 over GRE disertai aksi redistribute static.
* **Hasil yang Diharapkan:** Terkoneksi ke ISP; Dapat menjangkau internet luar; Terbentuknya rute inter-site dinamis OSPF.
* **Bukti yang Dikumpulkan:**

#### Screenshot `get system interface physical`
<img width="1275" height="1014" alt="image" src="https://github.com/user-attachments/assets/93ee0c04-03b0-4304-b61b-10a832eb6396" />


#### Screenshot `get router info routing-table all`
<img width="1274" height="532" alt="image" src="https://github.com/user-attachments/assets/c7f64cc5-9bfe-4d29-bcbc-0976382ae73f" />


#### Screenshot Firewall Policy (NAT Enabled)
<img width="1268" height="838" alt="image" src="https://github.com/user-attachments/assets/e73dfab8-61dc-493d-9b34-bcad9c95cc9b" />
<img width="1229" height="791" alt="image" src="https://github.com/user-attachments/assets/67c05c41-735c-4276-91e9-c450ad8ad531" />


#### Screenshot Ping dari FortiGate Jakarta ke Internet (`8.8.8.8`)
<img width="772" height="435" alt="image" src="https://github.com/user-attachments/assets/769ae667-1060-4bcd-ab5c-71dc2218dea3" />


#### Screenshot Ping ke IP Tunnel Surabaya (`172.16.0.2`)
<img width="772" height="435" alt="image" src="https://github.com/user-attachments/assets/0fb76c75-8ced-4d54-bdbc-4bd1c56c816f" />


#### Screenshot `get router info ospf neighbor`
<img width="1284" height="291" alt="image" src="https://github.com/user-attachments/assets/4e4d8e52-39fd-4a9a-be8a-80c97cb6ccbc" />


#### Screenshot `get router info routing-table ospf`
<img width="1284" height="291" alt="image" src="https://github.com/user-attachments/assets/51e640ed-2c9a-4e38-9ee2-159728158f89" />


---

### Tugas Modul 6 — Konfigurasi MikroTik ISP
* **Hal yang Dikonfigurasi:** IP address pada ether2 (ke JKT) dan ether3 (ke SBY); Aktivasi DHCP Client pada ether1 kearah Cloud PNETLab; Default route kearah cloud; Konfigurasi Firewall NAT masquerade luar pada out-interface ether1.
* **Hasil yang Diharapkan:** Menjadi jembatan transit WAN; Menyediakan akses NAT internet luar; Tidak menjalankan protokol OSPF internal enterprise.
* **Bukti yang Dikumpulkan:**

#### Screenshot `/ip address print`
<img width="1272" height="859" alt="image" src="https://github.com/user-attachments/assets/4cd5ffa8-23a0-4259-b9b6-136edbaab1c4" />


#### Screenshot `/ip route print`
<img width="1272" height="859" alt="image" src="https://github.com/user-attachments/assets/14adb831-0b50-4ffc-85fa-55c40dfd5af6" />


#### Screenshot `/ip firewall nat print`
<img width="1272" height="859" alt="image" src="https://github.com/user-attachments/assets/b61b22dd-6b51-4ad2-81a0-051858f705be" />


#### Screenshot Ping dari ISP ke Internet (`8.8.8.8`)
<img width="1272" height="859" alt="image" src="https://github.com/user-attachments/assets/50e8634f-23df-4fbc-b98d-fd8a3ce040ef" />


#### Screenshot Ping Antar-WAN FortiGate (JKT $\leftrightarrow$ SBY) via ISP


---

### Tugas Modul 7 — Konfigurasi Switch dan MikroTik Surabaya
* **Hal yang Dikonfigurasi:** Pembuatan VLAN 30 dan 40 di Switch Surabaya beserta port access/trunk; Pembuatan VLAN interface di MikroTik Surabaya; Konfigurasi IP gateway; Aktivasi DHCP Server lokal untuk VLAN 30 Sales; Setup IP static manual untuk VLAN 40 Operations; Default route ke FortiGate Surabaya.
* **Hasil yang Diharapkan:** Segmen cabang Surabaya terbentuk; Client VLAN 30 mendapat IP dinamis; Seluruh client Surabaya terhubung ke gateway transit.
* **Bukti yang Dikumpulkan:**

#### Screenshot `show vlan brief` (Switch Surabaya)
<img width="747" height="222" alt="image" src="https://github.com/user-attachments/assets/7d9d0faa-82e6-41ae-b435-0f7431b4ce72" />


#### Screenshot `show interfaces trunk` (Switch Surabaya)
<img width="712" height="267" alt="image" src="https://github.com/user-attachments/assets/b95aa3a6-451c-4b43-8533-09aed80a64e7" />


#### Screenshot `/ip address print` (MikroTik Surabaya)
<img width="608" height="193" alt="image" src="https://github.com/user-attachments/assets/c25093af-e0ba-45bd-ac2c-14260b9b2f5e" />


#### Screenshot `/ip dhcp-server print` & `/ip pool print`
<img width="820" height="86" alt="image" src="https://github.com/user-attachments/assets/54005f5c-6cd9-4183-a126-838cac1b8eaf" />
<img width="823" height="70" alt="image" src="https://github.com/user-attachments/assets/5c833366-e5bd-4917-ae59-6e2ae2a02321" />


#### Screenshot `/ip route print` (MikroTik Surabaya)
<img width="676" height="182" alt="image" src="https://github.com/user-attachments/assets/994c8d10-9c6b-449d-890d-729464d7fc6e" />


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
