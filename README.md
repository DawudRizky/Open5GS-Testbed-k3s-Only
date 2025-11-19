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

## Verifikasi Deployment

### 1. Cek Status Semua NF
Verifikasi semua pod Open5GS sudah berjalan dengan baik:
```bash
# List semua pods dengan detail
kubectl get pods -n open5gs -o wide
```

![NF status](https://drive.google.com/uc?id=1NqSwi3yVUfGkxtt-thxvHnzjjzMQrS_A)

Cek log untuk NF tertentu jika diperlukan:
```bash
# Check logs untuk NF tertentu
kubectl logs -n open5gs amf-0
kubectl logs -n open5gs smf-0
kubectl logs -n open5gs upf-0
```

---

### 2. Verifikasi Static IP Assignment
Jalankan script verifikasi untuk memastikan setiap NF mendapat static IP sesuai konfigurasi:
```bash
# Run verification script
sudo ./verify-static-ips.sh
```

Output yang diharapkan:
```
✓ nrf-0: 10.10.0.10
✓ scp-0: 10.10.0.200
✓ amf-0: 10.10.0.5
✓ smf-0: 10.10.0.4
✓ upf-0: 10.10.0.7
... (semua NF dengan IP yang sesuai)
```

![Verify static IP](https://drive.google.com/uc?id=1Lsm1NGyMh4egHM0vHMZjY7YNcM2k9hAJ)

---

### 3. Verifikasi MongoDB Connectivity

#### Pre-requisite: Cache MongoDB Image
Sebelum menjalankan script verifikasi, pastikan image MongoDB sudah ter-cache di K3s untuk menghindari timeout saat pulling image:
```bash
# Pull dan import MongoDB image ke K3s containerd
sudo ctr -n k8s.io images pull docker.io/library/mongo:5.0

# Verifikasi image sudah tersedia
sudo crictl images | grep mongo
```

#### Konfigurasi Script
Ubah `MONGO_IP` pada script `open5gs/open5gs-k3s-calico/verify-mongodb.sh` sesuai dengan IP host Anda:
```bash
sudo nano verify-mongodb.sh
# Ubah MONGO_IP="192.168.14.137" dengan IP host Anda
```

#### Jalankan Verifikasi
```bash
# Run MongoDB verification
sudo ./verify-mongodb.sh
```

Output yang diharapkan:
```
=== MongoDB Connectivity Test ===
Test 1: Network connectivity to 192.168.14.137:27017
✓ Port 27017 is reachable

Test 2: MongoDB authentication
✓ Connected successfully

Test 3: Testing from within K3s cluster...
✓ MongoDB is accessible from within K3s cluster
```

![Verify mongodb connection](https://drive.google.com/uc?id=1ngGPH-N-UD2Z3Up44DJYhp63j2a0CYTw)

---

### 4. Cek Service Connectivity

#### Catatan Penting: Open5GS Menggunakan HTTP/2
Open5GS SBI (Service Based Interface) hanya mendukung **HTTP/2** secara langsung. Saat melakukan testing dengan `curl`, **harus** menggunakan flag `--http2-prior-knowledge`. 

**Perintah yang salah:**
```bash
# ✗ Akan gagal dengan exit code 52 (Empty reply from server)
curl http://nrf:7777/nnrf-nfm/v1/nf-instances
curl --http2 http://nrf:7777/nnrf-nfm/v1/nf-instances
```

**Perintah yang benar:**
```bash
# ✓ Menggunakan HTTP/2 langsung
curl --http2-prior-knowledge http://nrf:7777/nnrf-nfm/v1/nf-instances
```

#### Test NF Connectivity
Verifikasi konektivitas antar NF menggunakan NRF API:
```bash
# Test dari AMF pod ke NRF
kubectl exec -it -n open5gs amf-0 -- /bin/bash

# Di dalam pod:
curl --http2-prior-knowledge http://nrf:7777/nnrf-nfm/v1/nf-instances
```

Atau test langsung dari NRF pod:
```bash
# Test NRF API dari dalam NRF pod
kubectl exec -n open5gs nrf-0 -- \
    curl -s --http2-prior-knowledge http://nrf:7777/nnrf-nfm/v1/nf-instances
```

Output yang diharapkan berupa JSON response dengan daftar NF yang terdaftar:
```json
{
  "_links": {
    "item": [
      {"href": "http://10.10.0.10:7777/nnrf-nfm/v1/nf-instances/..."},
    ],
    "totalItemCount": 9
  }
}
```

![Check service connectivity](https://drive.google.com/uc?id=1lsyyS3j8zTSlwxKA6uRiIL75Ge8LSdj1)

---

## Tugas 1: Konektivitas Dasar

### Objective

Verify bahwa Open5GS deployment berfungsi dengan benar dan dapat connect dengan UERANSIM.

### Prerequisites
- K3s deployment selesai
- Semua pods running
- UERANSIM binary tersedia

### Langkah-Langkah

#### 1.1 Persiapkan UERANSIM pada Host Eksternal

Navigasi ke direktori UERANSIM:
```bash
# Di mesin yang berbeda dari K3s (atau terminal baru dengan user biasa)
cd ~/Open5GS-Testbed/ueransim
```

**Modifikasi gNB Config:**

1. Dapatkan IP address host dan AMF pod:
```bash
# Cek IP address host
ip addr

# Cek IP address AMF pod
kubectl get pod amf-0 -n open5gs -o wide
```

![look for host's IP](https://drive.google.com/uc?id=1VCuvkoGtpC5SbvN5Yh-Xgq4_tZxdyuVR)
![look for amf pod's IP](https://drive.google.com/uc?id=1qOcIZ4t-bWd-eu3ZYipLjrb_cpcNfb1J)

2. Edit file `ueransim/configs/open5gs-gnb-k3s.yaml` dengan perubahan berikut:

**a. Ubah semua gNB interfaces menggunakan IP host:**

![modified gNB interfaces IP](https://drive.google.com/uc?id=1VTxtYT9YlRjkypaF5CXstBCVMEFaFiGd)

**b. Ubah AMF address ke IP AMF pod:**

![modified amfConfigs](https://drive.google.com/uc?id=1iNhtLFpN28bnESQ2rQrv0mCQvL5Od4Ar)

**Catatan:** gNB harus binding ke interface host karena berjalan langsung di host (bukan di dalam K3s cluster). Jika menggunakan pod IP, gNB akan gagal binding dengan error "Cannot assign requested address".

---

#### 1.2 Start gNB Simulator

Jalankan gNB simulator:
```bash
# Terminal 1 - gNB
cd ~/Open5GS-Testbed/ueransim
./build/nr-gnb -c configs/open5gs-gnb-k3s.yaml
```

Output yang diharapkan:

![gNB simulator running correctly](https://drive.google.com/uc?id=1r81KYb-bvb1wJsF-tboxndxInz07R59j)

---

#### 1.3 Start UE Simulator

**Modifikasi UE Config:**

1. Dapatkan IP address host (jika belum):
```bash
# Cek IP address host
ip addr
```

![look for host's IP](https://drive.google.com/uc?id=1Y_wmfyxo4GmyB-Hd-4gs7FVasdiQsC_s)

2. Edit file `ueransim/configs/open5gs-gnb-k3s.yaml`:

**Ubah gnbSearchList menggunakan IP host:**

![modified gNBSearchlist](https://drive.google.com/uc?id=11u8lzLqJgwGNvb6GEyIUs4s7ty8WA8v7)

**Catatan:** UE perlu mencari gNB menggunakan IP host karena gNB binding ke interface host. Jika menggunakan localhost atau IP lain, UE akan gagal menemukan cell ("no cell in coverage").

3. Jalankan UE simulator:
```bash
# Terminal 2 - UE
cd ~/Open5GS-Testbed/ueransim
sudo ./build/nr-ue -c configs/open5gs-ue-embb.yaml
```

Output yang diharapkan:

![UE simulator running correctly](https://drive.google.com/uc?id=19FR-0PNWKD2BIdNiK4OiQF8HYYrD3VDE)

---

#### 1.4 Test Basic Connectivity

Buka terminal baru untuk melakukan testing:
```bash
# Terminal 3 - Testing

# Test UE TUN interface
ip addr show uesimtun0
```

![UE TUN interface tested](https://drive.google.com/uc?id=1iUk_M_vvG5Wt9GxH_iq_vI_oa9eZTifI)

```bash
# Test gateway connectivity (UE -> UPF)
ping -I uesimtun0 -c 4 10.45.0.1
```

![gateway connectivity (UE -> UPF) tested](https://drive.google.com/uc?id=1VCdsllmpviIfT0d_-IsxdNmi4Tg4Jhp0)

```bash
# Test internet connectivity
ping -I uesimtun0 -c 4 8.8.8.8
```

![internet connectivity tested](https://drive.google.com/uc?id=1wbJfGhoXYl96B3ljoqFy4jjKVYi1o5ML)

```bash
# Test DNS resolution
nslookup google.com 8.8.8.8
```

![DNS resolution tested](https://drive.google.com/uc?id=1uTsXHtV-QhIRD4Qz_t3Fr8K2BjwqhmvO)

```bash
# Test HTTP/HTTPS
curl --interface uesimtun0 -I https://www.google.com
```

![HTTP/HTTPS tested](https://drive.google.com/uc?id=1PbL5XlL6dcx8-S4XzhCpjXSo2K2dSyqD)

---

#### 1.5 Dokumentasi Hasil

## Tugas 1: Konektivitas Dasar

**Tanggal**: [18/11/2025]
**Nama**: [Dawud Rizky Arianto | Nickolas Quinn Budiyono | Ghufron Bagaskara | Muhammad Danish Alfattah Lubis]
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

#### 1. SCP Cannot Ping NRF
**Problem:** SCP pod tidak memiliki utilitas `ping` yang diperlukan untuk testing konektivitas jaringan antar NF.

**Error Message:**
```
/bin/sh: ping: not found
```

#### 2. MongoDB Connection from K3s Cluster Fails
**Problem:** Pod Open5GS di dalam K3s cluster tidak dapat terhubung ke MongoDB yang berjalan di host.

**Error Message:**
```
MongoDB connection failed: Connection refused
```

**Root Cause:** MongoDB default binding hanya ke `127.0.0.1` (localhost), sehingga tidak dapat diakses dari pod K3s.

#### 3. UERANSIM gNB Binary Failed to Start
**Problem:** Binary UERANSIM gNB gagal start karena missing SCTP library dependency.

**Error Message:**
```
error while loading shared libraries: libsctp.so.1: cannot open shared object file
```

**Root Cause:** Library SCTP belum terinstall di host system.

#### 4. gNB Failing to Bind to Interfaces
**Problem:** gNB simulator gagal binding ke interfaces yang dikonfigurasi (`linkIp`, `ngapIp`, `gtpIp`, `gtpAdvertiseIp`).

**Error Message:**
```
[ERROR] Cannot assign requested address
```

**Root Cause:** gNB config menggunakan IP address pod K3s, padahal gNB berjalan langsung di host (bukan di dalam cluster). Interface binding harus menggunakan IP address host.

#### 5. UE Cannot Find Any Cells in Coverage
**Problem:** UE simulator gagal menemukan cell dari gNB.

**Error Message:**
```
[rrc] [error] Cell search failed, no cell in coverage
```

**Root Cause:** UE config `gnbSearchList` tidak sesuai dengan IP address dimana gNB sebenarnya binding (host IP).

#### 6. curl Error 52 When Testing NRF API
**Problem:** Saat testing NRF API dengan curl standard, mendapat error "Empty reply from server".

**Error Message:**
```
command terminated with exit code 52
```

**Root Cause:** Open5GS SBI menggunakan HTTP/2 secara eksklusif. Curl standard mengirim HTTP/1.1 yang ditolak oleh NRF.

#### 7. MongoDB Verification Timeout
**Problem:** Script `verify-mongodb.sh` timeout saat menjalankan test dari dalam K3s cluster.

**Root Cause:** Image `mongo:5.0` (275MB) belum ter-cache di K3s containerd, menyebabkan timeout saat pulling image.

---

### Resolution

#### 1. Install Ping Utility in SCP Container
Tambahkan `iputils-ping` ke dalam Dockerfile SCP:
```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        open5gs-scp \
        iputils-ping \
        curl && \
    apt-get clean
```

Rebuild dan reimport image:
```bash
cd ~/Open5GS-Testbed/open5gs/open5gs-k3s-calico
sudo ./build-import-containers.sh
```

#### 2. Configure MongoDB to Accept External Connections
Edit konfigurasi MongoDB untuk binding ke semua interface:
```bash
sudo nano /etc/mongod.conf
```

Ubah `bindIp`:
```yaml
net:
  port: 27017
  bindIp: 0.0.0.0  # Ubah dari 127.0.0.1
```

Restart MongoDB:
```bash
sudo systemctl restart mongod
```

#### 3. Install SCTP Library on Host
Install library SCTP yang diperlukan UERANSIM:
```bash
sudo apt-get update && sudo apt-get install -y libsctp1 lksctp-tools
```

Verifikasi instalasi:
```bash
ldconfig -p | grep sctp
```

#### 4. Configure gNB to Use Host IP Address
Edit file `ueransim/configs/open5gs-gnb-k3s.yaml`:
```yaml
linkIp: 192.168.14.137    # Ganti dengan IP host Anda
ngapIp: 192.168.14.137    # Ganti dengan IP host Anda  
gtpIp: 192.168.14.137     # Ganti dengan IP host Anda
gtpAdvertiseIp: 192.168.14.137  # Ganti dengan IP host Anda

amfConfigs:
  - address: 10.10.0.5    # IP address AMF pod di K3s
    port: 38412
```

#### 5. Configure UE gnbSearchList to Use Host IP
Edit file `ueransim/configs/open5gs-ue-embb.yaml`:
```yaml
gnbSearchList:
  - 192.168.14.137    # Ganti dengan IP host Anda (sama dengan gNB binding)
```

#### 6. Use Correct HTTP/2 Flag for curl
Gunakan flag `--http2-prior-knowledge` saat testing Open5GS SBI API:
```bash
# Correct command
curl --http2-prior-knowledge http://nrf:7777/nnrf-nfm/v1/nf-instances

# Wrong commands (will fail with exit code 52)
curl http://nrf:7777/nnrf-nfm/v1/nf-instances
curl --http2 http://nrf:7777/nnrf-nfm/v1/nf-instances
```

#### 7. Pre-cache MongoDB Image Before Verification
Pull dan cache image MongoDB sebelum menjalankan script verifikasi:
```bash
# Pull image ke K3s containerd
sudo ctr -n k8s.io images pull docker.io/library/mongo:5.0

# Verifikasi image tersedia
sudo crictl images | grep mongo

# Sekarang jalankan script verifikasi
sudo ./verify-mongodb.sh
```