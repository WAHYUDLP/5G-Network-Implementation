# 5G-Network-Implementation
Program Studi: Sarjana Ilmu Komputer
Topik: Implementasi dan Testing 5G Core Network
Tahun Akademik: 2025
Repository: https://github.com/rayhanegar/Open5GS-Testbed dan https://hackmd.io/@bNHBr_ClRs-kkD7KxWjBLQ/SywnPkJxWx#tugas-1-konektivitas-dasar

## Tugas 1: Konektivitas Dasar

**Tanggal**: [13 NOVEMBER 2025]
**Nama**: [WAHYU DWI LAKSANA PUTRI, SYAFA SYAKIRA SHALSABILA, ZAQIA MAHADEWI]
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
