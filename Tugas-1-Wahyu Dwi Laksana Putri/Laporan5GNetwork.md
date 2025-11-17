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
 ini buat paste ss an nunjukkin berhasil

Penjelasan: 
```text
ini penjelasan ges
```

## Step 2: Setup K3s Environment dengan Calico

Navigate ke direktori K3s dan jalankan setup script:
![Uploading image.png…]()

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
 ini buat paste ss an nunjukkin berhasil

Penjelasan: 
```text
ini penjelasan ges
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
 ini buat paste ss an nunjukkin berhasil

Penjelasan: 
```text
ini penjelasan ges
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
 ini buat paste ss an nunjukkin berhasil

Penjelasan: 
```text
ini penjelasan ges
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
 ini buat paste ss an nunjukkin berhasil

Penjelasan: 
```text
ini penjelasan ges
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
 ini buat paste ss an nunjukkin berhasil

Penjelasan: 
```text
ini penjelasan ges
```

###3. Verifikasi MongoDB Connectivity
```bash

# Run MongoDB verification
sudo ./verify-mongodb.sh

# Expected output:
# MongoDB: CONNECTED
# Database: open5gs
# Collection count: X subscribers

```
*_Screenshot Verifikasi Deployment 3_*
 ini buat paste ss an nunjukkin berhasil

Penjelasan: 
```text
ini penjelasan ges
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

 ini buat paste ss an nunjukkin berhasil

Penjelasan: 
```text
ini penjelasan ges
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

 ini buat paste ss an nunjukkin berhasil

Penjelasan: 
```text
ini penjelasan ges
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

 ini buat paste ss an nunjukkin berhasil

Penjelasan: 
```text
ini penjelasan ges
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

 ini buat paste ss an nunjukkin berhasil

Penjelasan: 
```text
ini penjelasan ges
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

 ini buat paste ss an nunjukkin berhasil

Penjelasan: 
```text
ini penjelasan ges
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

