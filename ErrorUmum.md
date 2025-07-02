# 🛠️ Proxmox HA Cluster Troubleshooting Guide

Berikut adalah daftar error umum yang mungkin ditemui saat membangun atau menjalankan High Availability (HA) cluster di Proxmox, lengkap dengan cara penanganannya.

---

## ❌ 1. Storage 'ha-shared' is not online

### Penyebab:

* NFS server mati / tidak terhubung
* Mount NFS gagal

### Solusi:

* Cek NFS server aktif:

  ```bash
  systemctl status nfs-server
  ```
* Cek mount dari node Proxmox:

  ```bash
  showmount -e 192.168.0.129
  mount | grep ha-shared
  ```
* Remount secara manual:

  ```bash
  pvesm set ha-shared --disable && pvesm set ha-shared --enable
  ```

---

## ❌ 2. Unable to start service ct\:XXX / service in error state

### Penyebab:

* CT tidak bisa start karena storage HA belum siap

### Solusi:

* Jalankan manual:

  ```bash
  ha-manager remove ct:100
  pct start 100
  ha-manager add ct:100 --state started
  ```

---

## ❌ 3. `Unit nfs-server.service not found`

### Penyebab:

* Paket `nfs-kernel-server` belum terinstal

### Solusi:

```bash
apt update
apt install nfs-kernel-server -y
```

---

## ❌ 4. `Temporary failure resolving 'ftp.debian.org'`

### Penyebab:

* DNS tidak dikonfigurasi

### Solusi:

Edit file `/etc/resolv.conf`:

```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

Atau gunakan DNS lokal:

```bash
echo "nameserver 192.168.0.1" > /etc/resolv.conf
```

---

## ❌ 5. `time drift` / waktu tidak sinkron antar node

### Penyebab:

* NTP belum dikonfigurasi

### Solusi:

Install dan aktifkan Chrony:

```bash
apt install chrony -y
systemctl enable chrony --now
```

Cek sinkronisasi:

```bash
chronyc sources
```

---

## ❌ 6. CT tidak auto-start saat container hidup

### Penyebab:

* Docker container tidak diset restart policy

### Solusi:

```bash
docker update --restart unless-stopped <container_name>
```

Atau saat create container:

```bash
docker run --restart unless-stopped ...
```

---

## ❌ 7. Node HA mati menyebabkan loop relocate

### Penyebab:

* Proxmox HA tidak bisa fence node

### Solusi:

* Pastikan node mati benar-benar shut down, atau fence manual
* Jika loop terjadi, hapus dari HA:

```bash
ha-manager remove ct:100
```

Lalu masukkan ulang jika perlu:

```bash
ha-manager add ct:100 --state started
```

---

## ✅ Tips Debug HA

```bash
journalctl -u pve-ha-lrm -f
ha-manager status
pct status <CTID>
pvesm status
```

---

## 📝 Author

Elfan Tube (2025)

---

Semoga membantu bro! 🚑
