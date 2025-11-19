
## Instalasi dan Setup

### Step 1: Persiapan Sistem

#### 1. Update Sistem dan Install Dependencies
Lakukan update sistem dan install beberapa dependencies dasar yang diperlukan untuk testbed Open5GS.
```bash
sudo apt-get update
sudo apt-get upgrade -y
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
```

#### 2. Install Docker
Docker diperlukan untuk menjalankan beberapa komponen dalam container. Berikut langkah instalasinya:
```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
Tambahkan repository Docker:
```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```
Install paket Docker:
```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl start docker
```

#### 3. Install SCTP Library
Library SCTP diperlukan untuk komunikasi antara komponen 5G Core dan UERANSIM. Install dengan:
```bash
sudo apt-get update && sudo apt-get install -y libsctp1 lksctp-tools
```

#### 4. Install MongoDB
MongoDB digunakan sebagai database untuk Open5GS. Berikut langkah instalasinya:
Install GPG key dan repository MongoDB:
```bash
sudo apt-get install gnupg curl
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.2.list
```
Install MongoDB dan aktifkan service:
```bash
sudo apt-get update
sudo apt-get install -y mongodb-org
sudo systemctl start mongod
sudo systemctl enable mongod
```
Konfigurasi agar MongoDB dapat diakses dari luar (ubah bindIp):
```bash
sudo nano /etc/mongod.conf
# Ubah bindIp menjadi 0.0.0.0
sudo systemctl restart mongod
```

#### 5. Membuat Direktori Log
Direktori log diperlukan untuk menyimpan log dari Open5GS.
```bash
sudo mkdir -p /mnt/data/open5gs-logs
sudo chmod 777 /mnt/data/open5gs-logs
```

#### 6. Clone Repository
Clone repository testbed ke server Anda:
```bash
git clone https://github.com/rayhanegar/Open5GS-Testbed
```

---

## Step 2: Setup K3s Environment dengan Calico

### Step 2: Setup K3s Environment dengan Calico

#### 1. Navigasi ke Direktori K3s
Pindah ke direktori K3s yang berisi script setup:
```bash
cd ~/Open5GS-Testbed/open5gs/open5gs-k3s-calico
```

#### 2. Jalankan Setup Script
Pastikan script dapat dieksekusi dan jalankan setup:
```bash
chmod +x setup-k3s-environment-calico.sh
sudo ./setup-k3s-environment-calico.sh
```

Script akan melakukan beberapa konfigurasi otomatis:
- Install K3s (lightweight Kubernetes)
- Setup Calico CNI untuk networking
- Konfigurasi static IP pool (10.10.0.0/24)
- Setup persistent storage
- Enable SCTP kernel module
- Konfigurasi IP forwarding

#### 3. Verifikasi Instalasi K3s
Setelah setup selesai, verifikasi instalasi K3s dan status node:
```bash
# Cek status K3s
sudo systemctl status k3s

# Cek node Kubernetes
kubectl get nodes
```
Output yang diharapkan:
```
NAME        STATUS   ROLES           AGE   VERSION
<hostname>  Ready    control-plane   Xm    v1.2X.X
```

---

### Step 3: Build dan Import Container Images

Sebelum menjalankan build script, lakukan modifikasi berikut pada beberapa file agar proses build dan import berjalan lancar:

#### 1. Modifikasi build-import-containers.sh
Tambahkan `sudo` sebelum command docker pada file berikut:
`open5gs/open5gs-k3s-calico/build-import-containers.sh`

#### 2. Modifikasi Dockerfile untuk Ping Tools
Tambahkan instalasi ping tools pada setiap Dockerfile di direktori berikut:
`open5gs/open5gs-compose/*/Dockerfile`

Contoh perubahan:
```dockerfile
# Install system dependencies and Open5GS scp in a single layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends software-properties-common gnupg && \
    add-apt-repository ppa:open5gs/latest && \
    apt-get update && \
    apt-get install -y --no-install-recommends \
        open5gs-scp \
        open5gs-common \
        gosu \
        ca-certificates \
        netbase \
        iputils-ping \
        curl && \
    mkdir -p /var/log/open5gs /etc/open5gs/tls /etc/open5gs/custom /var/run/open5gs && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

#### 3. Modifikasi YAML: Ubah Address MongoDB
Ubah address MongoDB di file berikut dengan IP address host Anda:
- `open5gs/open5gs-compose/pcf/pcf.yaml`
- `open5gs/open5gs-compose/udr/udr.yaml`

Contoh perubahan:
```yaml
db_uri: mongodb://192.168.14.137:27017/open5gs
```

#### 4. Jalankan Build Script
Setelah modifikasi selesai, pastikan script dapat dieksekusi dan jalankan build untuk membuat dan mengimport image Open5GS:
```bash
chmod +x build-import-containers.sh
sudo ./build-import-containers.sh
```

#### 5. Verifikasi Image
Setelah proses build selesai, verifikasi image yang sudah diimport ke K3s:
```bash
sudo k3s crictl images
```

---

### Step 4: Deploy Open5GS ke K3s

Sebelum menjalankan deployment, lakukan modifikasi berikut pada beberapa file agar proses deployment berjalan lancar:

#### 1. Modifikasi deploy-k3s-calico.sh
Pastikan fungsi `deploy_pod` sudah benar untuk menghindari false positive error. Contoh cuplikan kode yang benar:
```bash
deploy_pod() {
    local name=$1
    local file=$2
    local label=$3

    POD_DEPLOY_START[$name]=$(get_timestamp_ms)
    print_info "Deploying $name..."

    kubectl apply -f "$file" &>/dev/null

    # Wait for pod to exist first (avoid race condition in parallel deployments)
    local retries=0
    until kubectl get pod -l "$label" -n open5gs &>/dev/null; do
        sleep 0.5
        retries=$((retries + 1))
        if [ $retries -gt 20 ]; then
            POD_DEPLOY_END[$name]=$(get_timestamp_ms)
            POD_READY_TIME[$name]=$(calc_duration ${POD_DEPLOY_START[$name]} ${POD_DEPLOY_END[$name]})
            print_error "$name pod failed to be created after ${POD_READY_TIME[$name]}ms"
            return 1
        fi
    done

    # Wait for pod to be ready (increased timeout for image pull)
    if kubectl wait --for=condition=ready pod -l "$label" -n open5gs --timeout=180s &>/dev/null; then
        POD_DEPLOY_END[$name]=$(get_timestamp_ms)
        POD_READY_TIME[$name]=$(calc_duration ${POD_DEPLOY_START[$name]} ${POD_DEPLOY_END[$name]})
        print_success "$name ready in ${POD_READY_TIME[$name]}ms"
        return 0
    else
        POD_DEPLOY_END[$name]=$(get_timestamp_ms)
        POD_READY_TIME[$name]=$(calc_duration ${POD_DEPLOY_START[$name]} ${POD_DEPLOY_END[$name]})
        print_error "$name failed to start after ${POD_READY_TIME[$name]}ms"
        return 1
    fi
}
```

#### 2. Modifikasi mongod-external.yaml
Ubah IP address MongoDB dengan IP address host Anda pada file berikut:
`open5gs/open5gs-k3s-calico/00-foundation/mongod-external.yaml`

Contoh perubahan:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb
  namespace: open5gs
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - port: 27017
    targetPort: 27017
---
apiVersion: v1
kind: Endpoints
metadata:
  name: mongodb
  namespace: open5gs
subsets:
- addresses:
  - ip: 192.168.14.137  # Ganti dengan IP MongoDB host Anda
  ports:
  - port: 27017
```

#### 3. Jalankan Deployment Script
Setelah modifikasi selesai, pastikan script dapat dieksekusi dan jalankan deployment:
```bash
chmod +x deploy-k3s-calico.sh
sudo ./deploy-k3s-calico.sh
```

Script akan melakukan:
- Membuat namespace `open5gs`
- Setup Calico IPPool
- Membuat service MongoDB
- Deploy Network Function (NF) sesuai urutan dependency
- Generate deployment report

#### 4. Monitor Proses Deployment
Pantau proses deployment untuk memastikan semua pod berjalan dengan baik:
```bash
kubectl get pods -n open5gs -w
```
Tunggu hingga semua pod berstatus `Running` (±2-3 menit).

#### 5. Verifikasi Deployment
Setelah deployment selesai, verifikasi semua pod sudah berjalan:
```bash
kubectl get pods -n open5gs
```
Output yang diharapkan (semua harus Running):
```
NAME      READY   STATUS    RESTARTS   AGE
nrf-0     1/1     Running   0          2m
scp-0     1/1     Running   0          2m
udr-0     1/1     Running   0          2m
udm-0     1/1     Running   0          2m
ausf-0    1/1     Running   0          2m
pcf-0     1/1     Running   0          2m
nssf-0    1/1     Running   0          2m
amf-0     1/1     Running   0          2m
smf-0     1/1     Running   0          2m
upf-0     1/1     Running   0          2m
```

---

![look for host's IP](https://drive.google.com/uc?id=1VCuvkoGtpC5SbvN5Yh-Xgq4_tZxdyuVR)
![look for amf pod's IP](https://drive.google.com/uc?id=1qOcIZ4t-bWd-eu3ZYipLjrb_cpcNfb1J)
![modified gNB interfaces IP](https://drive.google.com/uc?id=1VTxtYT9YlRjkypaF5CXstBCVMEFaFiGd)
![modified amfConfigs](https://drive.google.com/uc?id=1iNhtLFpN28bnESQ2rQrv0mCQvL5Od4Ar)

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