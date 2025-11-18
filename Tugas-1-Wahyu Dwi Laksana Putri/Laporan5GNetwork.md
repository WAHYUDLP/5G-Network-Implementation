# 5G-Network-Implementation
- Program Studi: Sarjana Ilmu Komputer
- Topik: Implementasi dan Testing 5G Core Network
- Tahun Akademik: 2025
- Repository: https://github.com/rayhanegar/Open5GS-Testbed dan https://hackmd.io/@bNHBr_ClRs-kkD7KxWjBLQ/SywnPkJxWx#tugas-1-konektivitas-dasar


Dokumentasi ini menjelaskan prasyarat, arsitektur, dan langkah-langkah instalasi untuk mendeploy testbed Open5GS menggunakan K3s dan Calico.

##  Prasyarat

### Hardware 

* **Spesifikasi:**
    * 6 CPU cores
    * 9 GB RAM
    * 50+ GB storage


### Software Requirements

* **Linux Distribution**
    * Ubuntu 24.02 LTS
    * Kernel 5.4+
* **Packages yang akan di-install:**
    * K3s (Kubernetes)
    * Docker/Containerd
    * kubectl
    * git
    * curl
    * Wireshark (untuk packet analysis)

### Pengetahuan Prasyarat

* Linux command line basics
* Networking fundamentals (IP, DNS, TCP/UDP)
* Docker concepts (containers, images)
* YAML syntax
* Basic understanding of REST APIs

---

## Akses Sistem

Pastikan punya sudo access
```bash
sudo whoami  # Should output "root"
```

Clone repository
```bash

git clone [https://github.com/rayhanegar/Open5GS-Testbed.git](https://github.com/rayhanegar/Open5GS-Testbed.git)
cd Open5GS-Testbed
```

## Arsitektur Sistem
### Diagram Arsitektur Keseluruhan

```text
┌─────────────────────────────────────────────────────────────────┐
│                      UERANSIM (External)                        │
│  ┌──────────────┐  N2(NGAP)      ┌──────────────────────────┐   │
│  │  nr-gnb      │◄──SCTP/38412──►│                          │   │
│  │ (gNodeB)     │                │    K3s Cluster           │   │
│  └────┬─────────┘                │    (Kubernetes)          │   │
│       │ N3 (GTP-U)               │                          │   │
│       │ UDP/2152                 │  ┌──────────────────┐    │   │
│       └──────────────────────────┼─►│      UPF         │    │   │
│            uesimtun0             │  │ (10.10.0.7)      │    │   │
│            (10.45.0.0/24)        │  └──────────────────┘    │   │
│                                  │                          │   │
│  ┌──────────────┐                │  ┌─────────────────────┐ │   │
│  │  nr-ue       │◄───────────────┼─►│  Control Plane NFs  │ │   │
│  │ (UE)         │                │  │  AMF, SMF, NRF,     │ │   │
│  └──────────────┘                │  │  AUSF, UDM, etc     │ │   │
│                                  │  └─────────────────────┘ │   │
│                                  │                          │   │
│                                  │  ┌──────────────────┐    │   │
│                                  │  │ MongoDB Service  │    │   │
│                                  │  │ (External)       │    │   │
│                                  │  └──────────────────┘    │   │
│                                  │                          │   │
│                                  └──────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Komponen K3s yang Di-Deploy

| Komponen | StatefulSet | IP Statis | Port | Fungsi |
| :--- | :--- | :--- | :--- | :--- |
| NRF | nrf-0 | 10.10.0.10 | 7777 | Service Discovery |
| SCP | scp-0 | 10.10.0.200 | 7777 | Routing |
| AMF | amf-0 | 10.10.0.5 | 7777, 38412 | UE Registration |
| SMF | smf-0 | 10.10.0.4 | 7777 | Session Management |
| UPF | upf-0 | 10.10.0.7 | 2152 | User Plane |
| UDM | udm-0 | 10.10.0.12 | 7777 | Subscriber Data |
| UDR | udr-0 | 10.10.0.20 | 7777 | Data Repository |
| AUSF | ausf-0 | 10.10.0.11 | 7777 | Authentication |
| PCF | pcf-0 | 10.10.0.13 | 7777 | Policy Control |
| NSSF | nssf-0 | 10.10.0.14 | 7777 | Slice Selection |


### Network Slice Configuration

| Slice | SST | DNN | Subnet | Gateway |
| :--- | :--- | :--- | :--- | :--- |
| eMBB | 1 | embb.testbed | 10.45.0.0/24 | 10.45.0.1 |
| URLLC | 2 | urllc.v2x | 10.45.1.0/24 | 10.45.1.1 |
| mMTC | 3 | mmtc.testbed | 10.45.2.0/24 | 10.45.2.1 |

## Instalasi dan Setup

## Step 1: Persiapan Sistem

```bash
# Update system
sudo apt-get update
sudo apt-get upgrade -y

# Install dependencies
sudo apt-get install -y \
    curl \
    git \
    iptables \
    iptables-persistent \
    net-tools \
    iputils-ping \
    traceroute \
    tcpdump \
    wireshark \
    wireshark-common

# Create log directories
sudo mkdir -p /mnt/data/open5gs-logs
sudo chmod 777 /mnt/data/open5gs-logs

```

*_Screenshot Step 1_*

<img width="500" alt="image" src="https://github.com/user-attachments/assets/de602306-4d75-45c3-8be1-47b6e114e1f7" />

Penjelasan: 
```text
Berdasarkan Langkah 1: Persiapan Sistem, setelah melakukan pembaruan sistem, instalasi dependensi, dan pembuatan direktori log menggunakan perintah sudo mkdir -p /mnt/data/open5gs-logs diikuti dengan sudo chmod 777 /mnt/data/open5gs-logs, keberhasilan dari langkah pembuatan direktori ini diverifikasi melalui dua perintah ls yang ditampilkan pada Screenshot Step 1. Perintah pertama, ls -ld /mnt/data/open5gs-logs, mengonfirmasi bahwa direktori /mnt/data/open5gs-logs telah berhasil dibuat (d di awal izin), dimiliki oleh user dan group root, dan memiliki izin drwxrwxrwx (izin 777 yang berarti user, group, dan others memiliki izin baca, tulis, dan eksekusi), yang menunjukkan bahwa perintah pembuatan dan pemberian izin log direktori telah berhasil dieksekusi sesuai rencana. Perintah kedua, ls -l /mnt/data, semakin memperkuat hasil tersebut dengan menampilkan daftar isi direktori /mnt/data dan menunjukkan keberadaan sub-direktori open5gs-logs dengan izin yang sama (drwxrwxrwx), yang secara keseluruhan memastikan persiapan sistem untuk direktori log sudah benar.
```

## Step 2: Setup K3s Environment dengan Calico

Navigate ke direktori K3s dan jalankan setup script:


```bash
cd ~/Open5GS-Testbed/open5gs/open5gs-k3s-calico

# Make script executable
chmod +x setup-k3s-environment-calico.sh

# Run setup
sudo ./setup-k3s-environment-calico.sh
```
Script akan melakukan:
✅ Install K3s (lightweight Kubernetes)
✅ Setup Calico CNI untuk networking
✅ Configure static IP pool (10.10.0.0/24)
✅ Setup persistent storage
✅ Enable SCTP kernel module
✅ Configure IP forwarding


### Verifikasi K3s Installation:
```bash
# Check K3s status
sudo systemctl status k3s

# Check nodes
kubectl get nodes

# Expected output:
# NAME        STATUS   ROLES           AGE   VERSION
# ubuntudesktop  Ready    control-plane   Xm    v1.2X.X
```

*_Screenshot Step 2_*

1.  <img width="500" alt="image" src="https://github.com/user-attachments/assets/fdac9637-0564-4dd6-86ae-855e10a0146e" />

2. <img width="500" alt="image" src="https://github.com/user-attachments/assets/06bbd368-edf8-40eb-a2be-e625ca4954d4" />

3. <img width="500" alt="image" src="https://github.com/user-attachments/assets/cbf1f1d3-1b5a-41f2-b740-2451188a42cb" />

4. <img width="500" alt="image" alt="image" src="https://github.com/user-attachments/assets/3886e47f-e61b-4caa-b78c-dc3f4f68ee9b" />

5. <img width="500" alt="image" src="https://github.com/user-attachments/assets/5c4d93a4-45ec-470e-b059-e353b30fa84e" />


Penjelasan: 
```text
Berdasarkan Langkah 2: Setup K3s Environment dengan Calico, setelah menavigasi ke direktori yang tepat dan menjalankan skrip setup-k3s-environment-calico.sh menggunakan sudo ./setup-k3s-environment-calico.sh, skrip berhasil melakukan semua tugas yang direncanakan, diawali dengan pemeriksaan dan konfirmasi bahwa semua required packages sudah terinstal (ditunjukkan pada Screenshot no 4). Keberhasilan instalasi dan konfigurasi ini divalidasi melalui beberapa screenshot verifikasi: Screenshot no 5 menunjukkan status layanan K3s (sudo systemctl status k3s) adalah active (running) dan telah enabled, yang merupakan indikasi utama instalasi K3s berhasil; sementara itu, Screenshot 2 menunjukkan hasil verifikasi cluster (kubectl get nodes dan pod Calico) dengan node ubuntudesktop berstatus Ready dengan role control-plane,master, dan calico-node-vn5p9 pod berstatus Running dan READY 1/1, memverifikasi fungsionalitas K3s dan networking Calico CNI. Akhirnya, Screenshot 4 menyajikan Ringkasan (Summary) dan bagian Verifikasi (Verification) dari skrip, di mana semua langkah, termasuk kernel modules dimuat, K3s terinstal, Calico CNI terinstal dan terkonfigurasi, Open5GS IPPool (10.10.0.0/24) terkonfigurasi, dan kubectl access verified, semuanya ditandai dengan [SUCCESS] yang memastikan bahwa penyiapan lingkungan K3s dengan Calico CNI untuk Open5GS telah selesai sepenuhnya dan siap digunakan.
```

## Step 3: Build dan Import Container Images
```bash
# Make script executable
chmod +x build-import-containers.sh

# Build Open5GS images
sudo ./build-import-containers.sh

# Verifikasi image
sudo k3s crictl images
```

*_Screenshot Step 3_*

1. <img width="500" alt="image" src="https://github.com/user-attachments/assets/78256bb2-e7c7-48b5-9e5b-10ad7c37a95b" />
2. <img width="500" alt="image" src="https://github.com/user-attachments/assets/5a1006df-2a5b-45a0-885f-f366b25fa140" />



Penjelasan: 
```text
Berdasarkan Langkah 3: Build dan Import Container Images, setelah memberikan izin eksekusi pada skrip build-import-containers.sh dan menjalankannya menggunakan sudo ./build-import-containers.sh, skrip berhasil melakukan pemeriksaan dan impor image Open5GS. Screenshot no 1 menunjukkan proses eksekusi skrip di mana ia berhasil mengkonfirmasi bahwa semua image Open5GS yang diperlukan (nrf, scp, udr, udm, ausf, pcf, nssf, amf, smf, dan upf) dengan tag :latest telah exists (ada) dalam sistem. Keberhasilan ini kemudian divalidasi dengan perintah sudo k3s crictl images pada Screenshot no 2, yang menampilkan daftar container images yang tersedia. Dalam daftar tersebut, terlihat jelas semua image Open5GS yang dibutuhkan (docker.io/library/open5gs-amf, open5gs-ausf, open5gs-nrf, dst.) telah tersedia dengan tag latest dan memiliki IMAGE ID dan SIZE yang valid, mengkonfirmasi bahwa proses build atau import container images Open5GS telah berhasil diselesaikan dan semua komponen core jaringan siap untuk di-deploy.
```

## Step 4: Deploy Open5GS ke K3s
```bash
# Make script executable
chmod +x deploy-k3s-calico.sh

# Deploy
sudo ./deploy-k3s-calico.sh

# Monitor deployment (di terminal baru)
kubectl get pods -n open5gs -w

```
Deployment akan:

✅ Create namespace open5gs
✅ Setup Calico IPPool
✅ Create MongoDB service
✅ Deploy NF dalam order yang benar (dependency management)
✅ Generate deployment report


Tunggu hingga semua pod running (~2-3 menit):
```bash
# Check all pods
kubectl get pods -n open5gs

# Expected output (all should be Running):
# NAME                  READY   STATUS    RESTARTS   AGE
# nrf-0                 1/1     Running   0          2m
# scp-0                 1/1     Running   0          2m
# udr-0                 1/1     Running   0          2m
# udm-0                 1/1     Running   0          2m
# ausf-0                1/1     Running   0          2m
# pcf-0                 1/1     Running   0          2m
# nssf-0                1/1     Running   0          2m
# amf-0                 1/1     Running   0          2m
# smf-0                 1/1     Running   0          2m
# upf-0                 1/1     Running   0          2m

```

*_Screenshot Step 4_*

<img width="500" alt="image" alt="1 tiga command cek nodes dan pods" src="https://github.com/user-attachments/assets/6f7b1999-00b8-4905-81db-d1ecafd4e3a8" />

Penjelasan: 
```text
Berdasarkan Langkah 4: Deploy Open5GS ke K3s, setelah memberikan izin eksekusi dan menjalankan skrip deploy-k3s-calico.sh yang bertanggung jawab untuk membuat namespace open5gs, menyiapkan IPPool Calico, dan men-deploy semua Network Function (NF) Open5GS serta layanan MongoDB di cluster K3s, keberhasilan deployment ini dikonfirmasi melalui dua perintah verifikasi yang ditunjukkan pada Screenshot. Pertama, hasil perintah kubectl get pods -n open5gs menunjukkan bahwa semua pods Open5GS yang terdiri dari amf-0, ausf-0, nrf-0, nssf-0, pcf-0, scp-0, smf-0, udm-0, udr-0, dan upf-0 berada dalam status Running dan memiliki kondisi READY 1/1 (siap), yang memverifikasi bahwa semua komponen jaringan inti telah berhasil diinstal dan berfungsi. Kedua, perintah kubectl get svc -n open5gs memverifikasi keberhasilan eksposur layanan dengan menampilkan semua layanan NF Open5GS yang relevan (amf, ausf, mongodb, nrf, nssf, pcf, scp, smf, udm, udr, upf) telah berhasil dibuat dengan TYPE ClusterIP yang menunjukkan mereka dapat diakses di dalam cluster K3s, di mana layanan amf juga memiliki EXTERNAL-IP 10.0.2.15, mengkonfirmasi semua langkah deployment telah selesai dengan sukses.
```

## Verifikasi Deployment
### 1. Cek Status Semua NF
```bash

# List semua pods dengan detail
kubectl get pods -n open5gs -o wide

# Check logs untuk NF tertentu
kubectl logs -n open5gs amf-0
kubectl logs -n open5gs smf-0
kubectl logs -n open5gs upf-0
```

*_Screenshot Verifikasi Deployment 1_*

<img width="500" alt="image" src="https://github.com/user-attachments/assets/a086ca01-50cf-47c2-8fd3-d722afdf576d" />


Penjelasan: 
```text
Berdasarkan Verifikasi Deployment, langkah pertama untuk mengecek status semua Network Function (NF) dilakukan dengan perintah kubectl get pods -n open5gs -o wide. Keberhasilan verifikasi ini dinilai dari hasil pada Screenshot yang menunjukkan bahwa semua pods Open5GS (amf-0, ausf-0, nrf-0, nssf-0, pcf-0, scp-0, smf-0, udm-0, udr-0, dan upf-0) memiliki status Running dan kondisi READY 1/1, yang mengindikasikan bahwa semua komponen jaringan inti telah berhasil di-deploy dan berjalan dengan baik. Selain itu, opsi -o wide memberikan detail tambahan yang memverifikasi bahwa semua pods telah dialokasikan alamat IP dalam range yang telah dikonfigurasi (10.10.0.0/24) dan semuanya di-deploy pada node ubuntudesktop, menegaskan konsistensi dan integritas deployment di lingkungan K3s sebelum melanjutkan ke pemeriksaan logs NF spesifik.
```

### 2. Verifikasi Static IP Assignment
```bash

# Run verification script
sudo ./verify-static-ips.sh

# Expected output:
# ✓ nrf-0: 10.10.0.10
# ✓ scp-0: 10.10.0.200
# ✓ amf-0: 10.10.0.5
# ✓ smf-0: 10.10.0.4
# ✓ upf-0: 10.10.0.7
# ... (semua NF)

```
*_Screenshot Verifikasi Deployment 2_*

<img width="500" alt="image" src="https://github.com/user-attachments/assets/ed45577a-04a1-4020-a5c2-0002c405aa00" />
<img width="500" alt="image" src="https://github.com/user-attachments/assets/62af6424-ee5e-4766-bb8d-158784fd8aa1" />


Penjelasan: 
```text
Berdasarkan Verifikasi Static IP Assignment, setelah menjalankan skrip sudo ./verify-static-ips.sh yang bertujuan untuk memastikan fungsionalitas Calico CNI dan konfigurasi IP statis pada deployment Open5GS, hasilnya menunjukkan keberhasilan parsial dengan adanya masalah konektivitas. Pada Screenshot 1, skrip berhasil mengonfirmasi bahwa Calico terinstal, namespace open5gs ada, IPPool 'open5gs-pool' dengan CIDR 10.10.0.0/24 ada, dan yang terpenting, semua pods Open5GS telah berhasil mendapatkan alokasi alamat IP statis yang sesuai dengan rentang IPPool, seperti nssf-0: 10.10.0.14, amf-0: 10.10.0.5, dan scp-0: 10.10.0.200, memverifikasi bahwa penugasan IP statis telah sukses. Namun, terdapat adanya Some issues found selama pengujian konektivitas dan registrasi NF: SCP cannot ping NRF dan SCP cannot reach NRF HTTP, meskipun verifikasi menunjukkan 10 ConfigMaps hadir dan konfigurasi NRF serta AMF menggunakan IP statis yang benar. Meskipun ada masalah konektivitas antar-NF, konfirmasi bahwa 8 NF terdaftar dengan NRF dan penugasan IP statis pada semua pods telah berhasil tetap menunjukkan bahwa tujuan utama langkah verifikasi IP statis telah tercapai.
```

### 3. Verifikasi MongoDB Connectivity
```bash

# Run MongoDB verification
sudo ./verify-mongodb.sh

# Expected output:
# MongoDB: CONNECTED
# Database: open5gs
# Collection count: X subscribers

```
*_Screenshot Verifikasi Deployment 3_*

1. <img width="500" alt="image" src="https://github.com/user-attachments/assets/8be2926f-0e79-405e-8e11-29ccf8473e9d" />
2. <img width="500" alt="image" src="https://github.com/user-attachments/assets/5a4b61e4-9fe6-46d8-aefa-5ffe3c8520f7" />


Penjelasan: 
```text
Pada tahap 3. Verifikasi MongoDB Connectivity, dilakukan serangkaian pengujian untuk memastikan bahwa layanan MongoDB berjalan
dengan benar dan dapat diakses baik dari host maupun dari dalam cluster K3s. Proses ini dilakukan melalui skrip verify-mongodb.sh,
yang berfungsi sebagai prosedur diagnostik otomatis mencakup pengecekan konektivitas jaringan, autentikasi, serta akses internal
cluster. Pengujian dimulai dengan memastikan bahwa port 27017 pada instance MongoDB dapat dijangkau melalui koneksi TCP/IP, dan
hasil pada terminal (screenshot pertama, test 1) menunjukkan bahwa port tersebut berhasil diakses tanpa kendala. Selanjutnya,
skrip melakukan verifikasi autentikasi menggunakan mongo client dengan mencoba masuk ke database open5gs. Output (screenshot
pertama, test 2) menampilkan metadata seperti jumlah koleksi, storageSize, dan statistik indeks yang mengindikasikan kredensial
valid dan database dalam kondisi aktif serta responsif. Tahapan berikutnya adalah memastikan bahwa akses ke MongoDB juga dapat
dilakukan dari dalam cluster K3s. Untuk itu, skrip membuat sebuah pod sementara lalu mengeksekusi perintah mongo di dalam pod
tersebut. Pesan “MongoDB connection successful!” (screenshot kedua, test 3) menunjukkan bahwa jaringan internal cluster berfungsi
dengan benar dan pods memiliki rute serta izin akses yang diperlukan bagi modul seperti AMF, SMF, dan UDM yang bergantung pada
read/write database. Terakhir, skrip memeriksa log pod MongoDB untuk mendeteksi potensi error internal. Pesan “No errors found or
pod not ready” (screenshot kedua, test 4) menunjukkan tidak ditemukan gangguan yang berpotensi memengaruhi stabilitas layanan.
Dengan begitu, hasil verifikasi memastikan bahwa MongoDB terkonfigurasi dengan benar, dapat diakses dari berbagai lapisan sistem,
dan siap mendukung seluruh proses registrasi, autentikasi, serta manajemen sesi pada arsitektur 5G berbasis Open5GS.
```
### 4. Cek Service Connectivity
```bash

# Test NF connectivity from K3s pod
kubectl exec -it -n open5gs nrf-0 -- \
    /bin/sh -c "curl -s http://amf-0.amf.open5gs.svc.cluster.local:7777/nnrf-nfm/v1/nf-instances"

# Atau gunakan pod shell
kubectl exec -it -n open5gs amf-0 -- /bin/bash

# Di dalam pod:
$ curl -s http://nrf-0.nrf.open5gs.svc.cluster.local:7777/nnrf-nfm/v1/nf-instances
```

*_Screenshot Verifikasi Deployment 4_*

<img width="500" alt="image" src="https://github.com/user-attachments/assets/3ed8e57d-4535-4ec7-a45e-3563beead551" />

Penjelasan: 
```text
Pada tahap 4 Cek Service Connectivity, dilakukan proses verifikasi konektivitas antar Network Function (NF) di dalam cluster K3s
untuk memastikan bahwa komponen-komponen inti Open5GS dapat saling berkomunikasi melalui service internal Kubernetes. Proses
dimulai dengan mengeksekusi perintah kubectl exec untuk masuk ke dalam pod AMF menggunakan shell /bin/bash, sebagaimana terlihat
pada screenshot, dengan tujuan melakukan pengujian jaringan langsung dari dalam pod. Setelah berhasil masuk, dilakukan permintaan
HTTP menggunakan curl menuju endpoint NRF pada alamat http://nrf-0.nrf.open5gs.svc.cluster.local:7777/nnrf-nfm/v1/nf-instances.
Endpoint ini merupakan jalur standar yang digunakan untuk mendapatkan daftar NF yang terdaftar pada NRF melalui interface Nnrf_NFManagement.
Screenshot yang ditampilkan menunjukkan bahwa perintah curl dapat dijalankan tanpa error, menandakan bahwa AMF mampu melakukan
resolusi DNS internal cluster, menjangkau service NRF, dan menerima respons dari layanan tersebut. Keberhasilan ini memastikan bahwa
komunikasi antar-NF pada arsitektur Kubernetes telah berjalan sesuai mekanisme standar, di mana setiap layanan dapat diakses melalui
DNS internal dengan pola <pod>.<service>.<namespace>.svc.cluster.local. Dengan begitua AMF berhasil melakukan konektivitas ke NRF,
sehingga proses NF discovery dan registration dalam sistem Open5GS dapat berlangsung dengan normal dan mendukung operasional jaringan
5G pada testbed yang dikonfigurasi.
```

# Tugas 1: Konektivitas Dasar

## Objective

Verify bahwa Open5GS deployment berfungsi dengan benar dan dapat connect dengan 
UERANSIM.

## Prerequisites
- K3s deployment selesai
- Semua pods running
- UERANSIM binary tersedia

## Langkah-Langkah
### 1.1 Persiapkan UERANSIM pada host eksternal
```bash

# Di mesin yang berbeda dari K3s (atau terminal baru dengan user biasa):
cd ~/Open5GS-Testbed/ueransim

# Modifikasi gNB config untuk connect ke K3s AMF
# Ubah AMF address di open5gs-gnb-k3s.yaml:
#
# amfConfigs:
#   - address: <K3s_HOST_IP>  # IP address dari K3s cluster
#     port: 38412

```
*_Screenshot Langkah-Langkah 1.1_*

1. <img width="500" alt="image" src="https://github.com/user-attachments/assets/3dfc4993-0cce-4ec6-8ce9-59a616959be5" />
2. <img width="500" alt="image" src="https://github.com/user-attachments/assets/1ab39050-8eba-44c4-b4c3-2aa2f14aa0c3" />

Penjelasan: 
```text
Pada tahap 1.1 Persiapan UERANSIM pada Host Eksternal, dilakukan konfigurasi awal untuk memastikan bahwa elemen gNB yang
disimulasikan oleh UERANSIM dapat terhubung dengan core network Open5GS yang berjalan pada cluster K3s. Proses dimulai dengan
masuk ke direktori instalasi UERANSIM melalui perintah cd ~/Open5GS-Testbed/ueransim, kemudian membuka folder configs untuk
memverifikasi keberadaan berkas konfigurasi yang diperlukan. Screenshot pertama menunjukkan struktur direktori tersebut,
memastikan bahwa file open5gs-gnb-k3s.yaml tersedia sebagai berkas utama yang akan disesuaikan. Selanjutnya, file konfigurasi
tersebut dibuka untuk proses modifikasi, sebagaimana terlihat pada screenshot kedua yang menampilkan pengaturan identitas jaringan
(MCC, MNC, TAC, dan NCI), alamat IP lokal gNB untuk antarmuka N2 dan N3, serta bagian amfConfigs yang berisi informasi koneksi
menuju AMF. Pada tahap ini, penyesuaian penting dilakukan dengan mengubah nilai address pada bagian amfConfigs agar mengarah ke
alamat IP host K3s (<K3s_HOST_IP>) menggunakan port 38412 sebagai port default interface N2. Pembaruan ini memastikan bahwa gNB
dapat menemukan dan berkomunikasi dengan AMF, sehingga prosedur signalling serta pembentukan koneksi kontrol dalam arsitektur 5G
dapat berjalan dengan benar. 
```

## 1.2 Start gNB Simulator
```bash
# Terminal 1 - gNB
cd ~/Open5GS-Testbed/ueransim
./build/nr-gnb -c configs/open5gs-gnb-k3s.yaml

# Expected output:
# [sctp] [info] Trying to establish SCTP connection... (10.X.X.X:38412)
# [sctp] [info] SCTP connection established
# [ngap] [info] NG Setup procedure is successful

```
*_Screenshot Langkah-Langkah 1.2_*

<img width="500" alt="image" src="https://github.com/user-attachments/assets/4c8e0928-7677-4487-b154-e4eb1813250f" />


Penjelasan: 
```text
Pada tahap 1.2 Start gNB Simulator, dilakukan proses inisialisasi komponen gNB yang disimulasikan oleh UERANSIM untuk memastikan
bahwa elemen RAN dapat membangun koneksi kontrol dengan AMF yang berjalan pada cluster K3s. Proses dimulai dengan masuk ke direktori
instalasi UERANSIM (cd ~/Open5GS-Testbed/ueransim) dan menjalankan simulator gNB menggunakan perintah
./build/nr-gnb -c configs/open5gs-gnb-k3s.yaml. Sebelum eksekusi, dilakukan patching pada service AMF menggunakan kubectl untuk
menambahkan external IP agar AMF dapat diakses dari host eksternal tempat UERANSIM berjalan. Screenshot yang ditampilkan memperlihatkan
keseluruhan alur tersebut, di mana log terminal menunjukkan upaya gNB membangun koneksi SCTP ke AMF dan keberhasilan tahap ini ditandai
oleh pesan “SCTP connection established”, menunjukkan bahwa komunikasi pada interface N2 telah aktif. Tahapan berikutnya adalah proses
NG Setup, di mana AMF memberikan respons “NG Setup Response received” dan konfirmasi keberhasilan melalui pesan “NG Setup procedure is
successful”. Log selanjutnya memperlihatkan inisiasi RRC Setup, Initial Context Setup, serta persiapan sesi PDU pertama untuk UE, yang
menandakan bahwa gNB telah berfungsi penuh sebagai stasiun basis 5G dalam testbed. Secara keseluruhan, kombinasi instruksi teks dan
output terminal pada screenshot menunjukkan bahwa simulator gNB berhasil dijalankan, terhubung dengan benar ke network core Open5GS,
dan siap melayani koneksi UE pada arsitektur 5G yang telah dikonfigurasi.
```

## 1.3 Start UE Simulator
```bash

# Terminal 2 - UE
cd ~/Open5GS-Testbed/ueransim
sudo ./build/nr-ue -c configs/open5gs-ue-embb.yaml

# Expected output:
# [nas] [info] UE switches to state [MM-REGISTERED/NORMAL-SERVICE]
# [nas] [info] PDU Session establishment is successful PSI[1]
# [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.45.0.X] is up.
```
*_Screenshot Langkah-Langkah 1.3_*

<img width="500" alt="2 2 Logs UE bagian 1" src="https://github.com/user-attachments/assets/73bd96c4-a16b-4a6f-a3fd-c32aae39277e" />

<img width="500" alt="2 3 Logs UE bagian 2" src="https://github.com/user-attachments/assets/06368137-258d-400b-a8ca-5eb12c848755" />

Penjelasan: 
```text
Pada tahap 1.3 Start UE Simulator, proses inisialisasi perangkat pengguna (User Equipment/UE) dilakukan menggunakan UERANSIM
untuk menguji konektivitas terhadap core network Open5GS. Eksekusi perintah nr-ue dengan menggunakan konfigurasi
open5gs-ue-embb.yaml memulai simulasi UE dan memicu rangkaian prosedur attach serta registrasi ke jaringan 5G. Berdasarkan
output terminal, UE melalui tahapan PLMN search, RRC connection setup, autentikasi, security mode command, hingga initial
registration, yang seluruhnya diterima dan disetujui oleh jaringan hingga status berubah menjadi MM-REGISTERED/NORMAL-SERVICE.
Screenshot pertama menunjukkan fase awal ketika UE mendeteksi sel, melakukan koneksi RRC, serta memasuki mode layanan normal,
sedangkan screenshot kedua menampilkan kelanjutan proses yang mengonfirmasi bahwa Registration Accept telah diterima,
PDU Session Establishment berhasil (PSI[1]), dan antarmuka jaringan virtual uesimtun0 telah aktif dengan alamat IP
(misalnya 10.45.0.2). Dengan begitu, simulator UE telah resmi terhubung ke jaringan inti 5G dan siap digunakan untuk mengirim
maupun menerima trafik data dalam lingkungan testbed yang telah dikonfigurasi.
```

## 1.4 Test Basic Connectivity
```bash

#Terminal 3 - Testing
# Test UE TUN interface
ip addr show uesimtun0

# Test gateway connectivity (UE -> UPF)
ping -I uesimtun0 -c 4 10.45.0.1

# Test internet connectivity
ping -I uesimtun0 -c 4 8.8.8.8

# Test DNS resolution
nslookup google.com 8.8.8.8

# Test HTTP/HTTPS
curl --interface uesimtun0 -I https://www.google.com

```
*_Screenshot Langkah-Langkah 1.4_*

Test UE TUN interface dan Test gateway connectivity (UE -> UPF)
<img width="500"  alt="1. Test UI TUN dan gateway connectivity" src="https://github.com/user-attachments/assets/473022f4-9820-41d6-86c4-651226b67cab" />


 Test internet connectivity dan Test DNS resolution
<img width="500" alt="3 2 Test ping dan DNS resolution" src="https://github.com/user-attachments/assets/34428863-fd51-4b7f-9762-9dc2c2845509" />

 Test HTTP/HTTPS
<img width="500"  alt="3 3 Test HTTP-HTTPS" src="https://github.com/user-attachments/assets/0061b3ea-c361-4776-95d0-f3cca67caed0" />

 
Penjelasan: 
```text
Pada tahap 1.4 Test Basic Connectivity, dilakukan pengujian untuk memastikan bahwa User Equipment (UE) yang dijalankan
melalui UERANSIM tidak hanya berhasil ter-registrasi ke jaringan inti 5G, tetapi juga dapat mengirim dan menerima trafik
data melalui UPF hingga mencapai jaringan internet. Pengujian dimulai dengan memverifikasi antarmuka TUN (uesimtun0)
menggunakan perintah ip addr show, yang menunjukkan bahwa interface tersebut aktif dan telah memperoleh alamat IP (10.45.0.2),
menandakan keberhasilan pembentukan sesi PDU. Selanjutnya, konektivitas internal UE–UPF diuji menggunakan ping -I uesimtun0
10.45.0.1, yang memberikan respon ICMP tanpa packet loss, sehingga memastikan jalur komunikasi menuju gateway jaringan berfungsi
normal. Pengujian juga dilakukan ke konektivitas eksternal melalui ping -I uesimtun0 8.8.8.8, yang menunjukkan bahwa trafik dari
UE dapat mencapai internet publik melalui UPF. Layanan DNS divalidasi menggunakan nslookup google.com 8.8.8.8, yang berhasil
menerjemahkan nama domain ke alamat IP, menegaskan bahwa resolusi DNS berjalan dengan baik. Verifikasi pada lapisan aplikasi
dilakukan menggunakan curl --interface uesimtun0 https://www.google.com, yang menghasilkan respons HTTP/2 200 dari server Google,
 menandakan bahwa UE dapat mengakses layanan web secara penuh. Screenshot pertama memperlihatkan keberhasilan pengecekan interface
TUN dan konektivitas ke UPF, screenshot kedua menunjukkan bahwa UE dapat melakukan ping ke internet dan melakukan resolusi DNS,
sementara screenshot ketiga menampilkan respons HTTP/HTTPS yang menegaskan akses web berhasil dilakukan. Seluruh pengujian ini
mengonfirmasi bahwa konektivitas data UE berfungsi sepenuhnya, mulai dari komunikasi internal dalam jaringan 5G hingga akses ke
layanan internet dan aplikasi dalam lingkungan testbed yang dikonfigurasi. 
```


============================================================================================

# Dokumentasi Tugas 1: Konektivitas Dasar

**Tanggal**: [13 NOVEMBER 2025]

**Nama**: [WAHYU DWI LAKSANA PUTRI, SYAFA SYAKIRA SHALSABILLA, ZAQIA MAHADEWI]

**Status K3s**: [WORKING]

### gNB Registration
- Status: [SUCCESS]
- Time taken: [~172 ms (Start: 17:11:13.139 -> Success: 17:11:13.311)]
- AMF Connection: [ESTABLISHED(SCTP Connected to 10.0.2.15:38412)]

### UE Registration
- Status: [SUCCESS(MM-REGISTERED & PDU Session Established)]
- Time taken: [~3.06 detik (Start: 17:16:17.114 -> TUN Up: 17:16:20.176)]
- IMSI: [999700000000001 (Default Profile)]
- TUN Interface: [uesimtun0]
- IP Address: [10.45.0.2]

### Connectivity Tests
| Test | Result | RTT (ms) |
|------|--------|----------|
| UPF Gateway (10.45.0.1) | ✓ PASS | 10.23 ms (avg)	 |
| Internet (8.8.8.8) | ✓ PASS | 111.84 ms (avg)	 |
| DNS Resolution | ✓ PASS | - |
| HTTP/HTTPS | ✓ PASS | - |

### Issues Encountered
- SCTP Bind Error (gNB): Awalnya gNB gagal start dengan error Cannot assign requested address karena konfigurasi menggunakan IP default tutorial (10.34.4.130).
- Connection Refused (gNB -> AMF): gNB tidak bisa lapor ke AMF karena Kubernetes Service masih bertipe ClusterIP (tertutup) dan ExternalIP salah.
- No Cells in Coverage (UE): UE gagal menemukan sinyal karena mencari gNB di localhost (127.0.0.1), padahal gNB berjalan di IP Host.
- DNS Timeout: nslookup sempat mengalami timeout pada percobaan awal, namun berhasil pada percobaan berikutny

### Resolution
- IP Configuration: Mengubah seluruh konfigurasi IP pada open5gs-gnb-k3s.yaml dan open5gs-ue-embb.yaml menjadi IP Host (10.0.2.15).
- Service Patching: Melakukan patching pada service AMF di Kubernetes untuk mengubah tipe menjadi NodePort (Port 38412) dan menetapkan externalIPs ke 10.0.2.15.
- UE Search List: Mengupdate gnbSearchList pada konfigurasi UE agar mengarah ke IP gNB yang benar (10.0.2.15).


## Kesimpulan

