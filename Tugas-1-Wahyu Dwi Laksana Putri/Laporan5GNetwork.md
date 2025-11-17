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

<img width="1470" height="266" alt="image" src="https://github.com/user-attachments/assets/de602306-4d75-45c3-8be1-47b6e114e1f7" />


Penjelasan: 
```text
ini penjelasan ges
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

1.  <img width="2411" height="1209" alt="image" src="https://github.com/user-attachments/assets/fdac9637-0564-4dd6-86ae-855e10a0146e" />

2. <img width="2675" height="193" alt="image" src="https://github.com/user-attachments/assets/06bbd368-edf8-40eb-a2be-e625ca4954d4" />

3. <img width="1715" height="269" alt="image" src="https://github.com/user-attachments/assets/cbf1f1d3-1b5a-41f2-b740-2451188a42cb" />

4. <img width="2400" height="1381" alt="image" src="https://github.com/user-attachments/assets/3886e47f-e61b-4caa-b78c-dc3f4f68ee9b" />

5. <img width="1805" height="1429" alt="image" src="https://github.com/user-attachments/assets/5c4d93a4-45ec-470e-b059-e353b30fa84e" />


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

1. <img width="1791" height="696" alt="image" src="https://github.com/user-attachments/assets/78256bb2-e7c7-48b5-9e5b-10ad7c37a95b" />
2. <img width="2162" height="1157" alt="image" src="https://github.com/user-attachments/assets/5a1006df-2a5b-45a0-885f-f366b25fa140" />



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

<img width="2133" height="1265" alt="1 tiga command cek nodes dan pods" src="https://github.com/user-attachments/assets/6f7b1999-00b8-4905-81db-d1ecafd4e3a8" />

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

<img width="2432" height="584" alt="image" src="https://github.com/user-attachments/assets/a086ca01-50cf-47c2-8fd3-d722afdf576d" />


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

<img width="1516" height="1266" alt="image" src="https://github.com/user-attachments/assets/ed45577a-04a1-4020-a5c2-0002c405aa00" />
<img width="1521" height="1261" alt="image" src="https://github.com/user-attachments/assets/62af6424-ee5e-4766-bb8d-158784fd8aa1" />


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

1. <img width="1517" height="1233" alt="image" src="https://github.com/user-attachments/assets/8be2926f-0e79-405e-8e11-29ccf8473e9d" />
2. <img width="1442" height="1348" alt="Cuplikan layar 2025-11-13 232818" src="https://github.com/user-attachments/assets/5a4b61e4-9fe6-46d8-aefa-5ffe3c8520f7" />


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

<img width="1502" height="298" alt="image" src="https://github.com/user-attachments/assets/3ed8e57d-4535-4ec7-a45e-3563beead551" />

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

1. <img width="1523" height="399" alt="image" src="https://github.com/user-attachments/assets/3dfc4993-0cce-4ec6-8ce9-59a616959be5" />
2. <img width="1499" height="1392" alt="image" src="https://github.com/user-attachments/assets/1ab39050-8eba-44c4-b4c3-2aa2f14aa0c3" />

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

<img width="1610" height="1498" alt="2 1 Logs gNB" src="https://github.com/user-attachments/assets/4c8e0928-7677-4487-b154-e4eb1813250f" />


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

<img width="1606" height="1497" alt="2 2 Logs UE bagian 1" src="https://github.com/user-attachments/assets/73bd96c4-a16b-4a6f-a3fd-c32aae39277e" />

<img width="1692" height="1409" alt="2 3 Logs UE bagian 2" src="https://github.com/user-attachments/assets/06368137-258d-400b-a8ca-5eb12c848755" />

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

Test UE TUN interface dan Test gateway connectivity (UE -> UPF)
<img width="1619" height="1010" alt="3 1 Test UE TUN interface  dan gateway connectivity" src="https://github.com/user-attachments/assets/e8d52766-aa30-46a3-87d1-7dafa783a870" />


 Test internet connectivity dan Test DNS resolution
 <img width="1599" height="1135" alt="3 2 Test ping dan DNS resolution" src="https://github.com/user-attachments/assets/34428863-fd51-4b7f-9762-9dc2c2845509" />

 Test HTTP/HTTPS
<img width="1615" height="1179" alt="3 3 Test HTTP-HTTPS" src="https://github.com/user-attachments/assets/0061b3ea-c361-4776-95d0-f3cca67caed0" />

 
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

