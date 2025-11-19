![Testbed Topology](https://drive.google.com/uc?id=1VCuvkoGtpC5SbvN5Yh-Xgq4_tZxdyuVR)
![Testbed Topology](https://drive.google.com/uc?id=1qOcIZ4t-bWd-eu3ZYipLjrb_cpcNfb1J)
![Testbed Topology](https://drive.google.com/uc?id=1VTxtYT9YlRjkypaF5CXstBCVMEFaFiGd)
![Testbed Topology](https://drive.google.com/uc?id=1iNhtLFpN28bnESQ2rQrv0mCQvL5Od4Ar)

---

## Tugas 1: Konektivitas Dasar

**Tanggal**: [18/11/2025]
**Nama**: [DAWUD RIZKY ARIANTO]
**Status K3s**: [WORKING]

### gNB Registration
- Status: [SUCCESS]
- Time taken: [156 ms]
- AMF Connection: [ESTABLISHED]

### UE Registration
- Status: [SUCCESS]
- Time taken: [825 ms]
- IMSI: [imsi-001011000000001]
- TUN Interface: [uesimtun0]
- IP Address: [10.45.0.2]

### Connectivity Tests
| Test | Result | RTT (ms) |
|------|--------|----------|
| UPF Gateway (10.45.0.1) | ✓ PASS | 1.658 |
| Internet (8.8.8.8) | ✓ PASS | 33.333 |
| DNS Resolution | ✓ PASS | - |
| HTTP/HTTPS | ✓ PASS | - |

### Issues Encountered
- SCP cannot ping NRF
- Connection to MongoDB from within the K3s cluster fails
- UERANSIM gNB binary failed to start because the required SCTP library `libsctp.so.1` is missing at the host
- UERANSIM gNB is failing to bind to it's interfaces (`linkIp`, `ngapIp`, `gtpIp`, `gtpAdvertiseIp`)
- UERANSIM UE is failing to find any cells in coverage

### Resolution
- Install ping tool by adding `iputils-ping` into SCP's Dockerfile
- Change the bindIp setting in `mongod.conf` from `127.0.0.1` to `0.0.0.0` then restart MongoDB
- Install the SCTP library at the host: `sudo apt-get update && sudo apt-get install -y libsctp1 lksctp-tools`
- Modify `open5gs-ue-embb.yaml` by using host's main IP address for all gNB interfaces (`linkIp`, `ngapIp`, `gtpIp`, `gtpAdvertiseIp`)
- Modify `open5gs-ue-embb.yaml` by updating `gnbSearchList` to use host's main IP address