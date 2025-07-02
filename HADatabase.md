# 📘 Proxmox + HA Container + Galera Cluster (MySQL HA)

Dokumentasi lengkap proses setup High Availability di Proxmox dan Database HA menggunakan Galera Cluster.

---

## 🧱 Infrastruktur

* **Proxmox VE Node**: 3 node (pve1, pve2, pve3)
* **Storage HA**: NFS server (dari `192.168.0.129`, folder `/mnt/ha-shared`)
* **HA Container**: LXC (ct:100) dengan Docker dan project web
* **HA Database**: MariaDB (Galera Cluster) di 3 node DB (db1, db2, db3)

### 4. Buat Container (LXC) di NFS

```bash
pct create 100 local:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst \
  -storage ha-shared \
  -rootfs ha-shared:20 \
  -hostname webha \
  -net0 name=eth0,bridge=vmbr0,ip=dhcp \
  -features nesting=1 \
  -password yourpass

pct start 100
```

Masuk container dan setup Docker:

```bash
apt update && apt install docker.io docker-compose -y
```

Jalankan project kamu (Nginx, PHP, MariaDB, dll).
Pastikan Docker container pakai opsi `restart: always` di `docker-compose.yml` atau `--restart always` saat `docker run`.

---

### 5. Tambahkan Container ke HA Manager

```bash
ha-manager add ct:100 --state started
```

Verifikasi:

```bash
ha-manager status
```

---

### 6. Setup Galera Cluster (MariaDB HA)

#### Di semua node DB (db1, db2, db3):

```bash
apt install mariadb-server galera-4 rsync -y
```

Edit `/etc/mysql/mariadb.conf.d/60-galera.cnf`:

```ini
[mysqld]
binlog_format=ROW
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
bind-address=0.0.0.0

wsrep_on=ON
wsrep_provider=/usr/lib/galera/libgalera_smm.so

wsrep_cluster_name="mariadb-cluster"
wsrep_cluster_address="gcomm://192.168.0.201,192.168.0.202,192.168.0.203"

wsrep_node_name="dbX"
wsrep_node_address="192.168.0.20X"

wsrep_sst_method=rsync
```

#### Bootstrap dari db1:

```bash
systemctl stop mysql
rm -f /var/lib/mysql/grastate.dat
rm -f /var/lib/mysql/galera.cache
galera_new_cluster
```

Cek:

```bash
mysql -u root -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```

#### Join db2 dan db3:

```bash
systemctl stop mysql
rm -f /var/lib/mysql/grastate.dat
rm -f /var/lib/mysql/galera.cache
systemctl start mysql
```

---

### 7. Tes HA & Cluster

* Coba matikan satu Proxmox node, lihat apakah container pindah otomatis.
* Coba matikan satu DB node, lihat cluster size menyusut lalu kembali.

---

## ✅ Selesai

Kamu sekarang punya:

* High Availability Container di Proxmox
* High Availability Database dengan Galera Cluster

---

## 📁 Lanjutkan

Untuk dokumentasi error & troubleshooting: cek `errors.md`

Untuk dokumentasi teknis Docker project: buatkan `docker-compose.yml` di folder `webha/`
