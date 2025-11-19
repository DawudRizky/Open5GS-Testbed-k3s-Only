
## Instalasi dan Setup

### Step 1: Persiapan Sistem

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

# Install Docker
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl start docker

# Install SCTP library
sudo apt-get update && sudo apt-get install -y libsctp1 lksctp-tools

# Install MongoDB
sudo apt-get install gnupg curl

curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor

echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.2.list

sudo apt-get update

sudo apt-get install -y mongodb-org

sudo systemctl start mongod

sudo systemctl enable mongod

# Modifikasi bindIp di mongod.conf
sudo nano /etc/mongod.conf
# Ubah bindIp menjadi 0.0.0.0
sudo systemctl restart mongod

# Create log directories
sudo mkdir -p /mnt/data/open5gs-logs
sudo chmod 777 /mnt/data/open5gs-logs

# Clone this repo on your server
git clone https://github.com/rayhanegar/Open5GS-Testbed
```


![Testbed Topology](https://drive.google.com/uc?id=1VCuvkoGtpC5SbvN5Yh-Xgq4_tZxdyuVR)
![Testbed Topology](https://drive.google.com/uc?id=1qOcIZ4t-bWd-eu3ZYipLjrb_cpcNfb1J)
![Testbed Topology](https://drive.google.com/uc?id=1VTxtYT9YlRjkypaF5CXstBCVMEFaFiGd)
![Testbed Topology](https://drive.google.com/uc?id=1iNhtLFpN28bnESQ2rQrv0mCQvL5Od4Ar)

---

![Testbed Topology](https://drive.google.com/uc?id=1r81KYb-bvb1wJsF-tboxndxInz07R59j)

---

![Testbed Topology](https://drive.google.com/uc?id=1Y_wmfyxo4GmyB-Hd-4gs7FVasdiQsC_s)
![Testbed Topology](https://drive.google.com/uc?id=11u8lzLqJgwGNvb6GEyIUs4s7ty8WA8v7)
![Testbed Topology](https://drive.google.com/uc?id=19FR-0PNWKD2BIdNiK4OiQF8HYYrD3VDE)

---

![Testbed Topology](https://drive.google.com/uc?id=1iUk_M_vvG5Wt9GxH_iq_vI_oa9eZTifI)
![Testbed Topology](https://drive.google.com/uc?id=1VCdsllmpviIfT0d_-IsxdNmi4Tg4Jhp0)
![Testbed Topology](https://drive.google.com/uc?id=1wbJfGhoXYl96B3ljoqFy4jjKVYi1o5ML)
![Testbed Topology](https://drive.google.com/uc?id=1uTsXHtV-QhIRD4Qz_t3Fr8K2BjwqhmvO)
![Testbed Topology](https://drive.google.com/uc?id=1PbL5XlL6dcx8-S4XzhCpjXSo2K2dSyqD)

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