# Proxmox VE High Availability (HA) Cluster Setup with NFS Shared Storage

## 📌 Overview

Tutorial ini menjelaskan langkah-langkah lengkap membangun Proxmox VE Cluster dengan fitur High Availability (HA) untuk Container (CT) menggunakan shared storage berbasis NFS.

---

## 🧱 Topologi

* **3 Node Proxmox VE**:

  * `pve1` (192.168.0.152)
  * `pve2` (192.168.0.153)
  * `pve3` (192.168.0.154)
* **1 Server NFS**:

  * `nfs-server` (192.168.0.129) → menyediakan storage `/mnt/ha-shared`

---

## 🔧 Langkah-langkah Setup

### 1. Install Proxmox VE di ketiga node

Pastikan semua node Proxmox bisa saling ping dan SSH.

---

### 2. Buat Cluster di pve1

```bash
pvecm create mycluster
```

### 3. Join Node ke Cluster

Jalankan di `pve2` dan `pve3`:

```bash
pvecm add 192.168.0.152
```

---

### 4. Konfigurasi Storage NFS di Server NFS

Di server NFS (192.168.0.129):

```bash
mkdir -p /mnt/ha-shared
chmod 777 /mnt/ha-shared
apt update && apt install -y nfs-kernel-server
```

Edit `/etc/exports`:

```
/mnt/ha-shared 192.168.0.0/24(rw,sync,no_subtree_check,no_root_squash)
```

Lalu restart:

```bash
systemctl restart nfs-server
```

---

### 5. Tambahkan NFS Storage ke Semua Node Proxmox

Jalankan di masing-masing node (`pve1`, `pve2`, `pve3`):

```bash
pvesm add nfs ha-shared \
  --server 192.168.0.129 \
  --export /mnt/ha-shared \
  --content rootdir \
  --path /mnt/pve/ha-shared
```

---

### 6. Pindahkan CT ke Storage Shared (ha-shared)

Contoh untuk CT ID 100:

```bash
pct move 100 --storage ha-shared
```

---

### 7. Tambahkan CT ke HA Manager

```bash
ha-manager add ct:100 --state started
```

Cek status:

```bash
ha-manager status
```

---

### 8. Tes Failover

Matikan node tempat CT berjalan dan pastikan service berpindah ke node lain otomatis.

---

## 💡 Tips Tambahan

### Sinkronisasi Waktu

Pastikan semua node sinkron waktu:

```bash
apt install chrony -y
systemctl enable chrony --now
```

### Restart Docker Otomatis (jika pakai Docker di CT)

```bash
docker update --restart unless-stopped <container_id>
```

---

## 📁 Struktur Folder

```
/mnt/ha-shared/   <-- disediakan oleh server NFS dan di-mount semua node
```

---

## ✅ Status HA

Cek dengan:

```bash
ha-manager status
```

Output contoh:

```
service ct:100 (pve2, started)
```

---

## 📚 Referensi

* [https://pve.proxmox.com/wiki/High\_Availability\_Cluster\_4.x](https://pve.proxmox.com/wiki/High_Availability_Cluster_4.x)
* [https://pve.proxmox.com/wiki/Storage:\_NFS](https://pve.proxmox.com/wiki/Storage:_NFS)

---

## 📝 Author

Elfan Tube (2025)

---

Happy clustering! 🚀
