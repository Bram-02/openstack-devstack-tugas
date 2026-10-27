# Instalasi OpenStack dengan DevStack (Revisi 2)

Tugas instalasi OpenStack 2026.1 secara all-in-one memakai **DevStack** pada satu VM Ubuntu Server 24.04 di VMware Workstation, lengkap dengan deploy website sederhana pada instance OpenStack.

**Penyusun:** Bram Naibaho (NIM 43324002)

## Hasil

- OpenStack 2026.1 (DevStack) terpasang dan semua service inti aktif: Keystone, Glance, Nova, Placement, Neutron, dan Horizon.
- Dashboard Horizon dapat diakses dari browser Windows.
- Instance Ubuntu 24.04 (`web-server`) berjalan dengan floating IP, dan melayani website sederhana yang dapat dibuka dari browser.

## Spesifikasi lingkungan

| Komponen | Keterangan |
| --- | --- |
| Hypervisor host | VMware Workstation 17 (Windows) |
| OS VM | Ubuntu Server 24.04 LTS |
| vCPU / RAM / Disk | 2 / 4 GB / 40 GB (root LVM diperluas menjadi 38 GB) |
| Swap | 9.8 GiB (3.8 GiB bawaan + swapfile 6 GiB) |
| Jaringan VM | NAT, IP `192.168.23.160` |
| OpenStack | DevStack `stable/2026.1`, Neutron OVN, hypervisor QEMU |
| Instance website | image `ubuntu-24.04`, flavor `web` (1 vCPU, 512 MB, 5 GB), floating IP `172.24.4.2` |

## Isi repository

| Path | Keterangan |
| --- | --- |
| `Laporan_Instalasi_OpenStack_DevStack.docx` | Laporan lengkap (langkah, kendala, dan verifikasi) |
| `config/local.conf` | Konfigurasi DevStack (password diganti placeholder) |
| `config/web.yaml` | Skrip cloud-init untuk instance website (password diganti placeholder) |
| `website/index.html` | Halaman website yang dideploy |
| `screenshots/` | Screenshot bukti, bernomor sesuai gambar di laporan |

## Langkah singkat

1. Buat VM Ubuntu Server 24.04 (2 vCPU, 4 GB RAM, 40 GB disk). Aktifkan virtualisasi VT-x/AMD-V.
2. Pastikan root partition memakai seluruh disk. Jika installer hanya memakai separuh, perbesar:

   ```bash
   sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
   ```

3. Tambahkan swap bila RAM terbatas, lalu buat user `stack`:

   ```bash
   sudo useradd -s /bin/bash -d /opt/stack -m stack
   echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
   sudo -u stack -i
   ```

4. Clone DevStack dan jalankan instalasi:

   ```bash
   cd /opt/stack
   git clone https://opendev.org/openstack/devstack
   cd devstack
   git checkout stable/2026.1
   cp /path/ke/config/local.conf .   # isi ADMIN_PASSWORD dan HOST_IP
   ./stack.sh
   ```

5. Verifikasi:

   ```bash
   source openrc admin admin
   openstack service list
   openstack compute service list
   openstack network agent list
   ```

   Horizon: `http://<HOST_IP>/dashboard` (domain `default`, user `admin`).

## Deploy website pada instance

```bash
# image, flavor, security group (project demo)
openstack image create ubuntu-24.04 --file noble-server-cloudimg-amd64.img \
  --disk-format qcow2 --container-format bare --public
openstack flavor create web --vcpus 1 --ram 512 --disk 5
source openrc demo demo
openstack security group rule create --proto tcp --dst-port 80 default
openstack security group rule create --proto tcp --dst-port 22 default
openstack security group rule create --proto icmp default
openstack keypair create mykey > ~/mykey.pem && chmod 600 ~/mykey.pem

# instance + floating IP
openstack server create --flavor web --image ubuntu-24.04 --network private \
  --key-name mykey --user-data config/web.yaml web-server
openstack floating ip create public
openstack server add floating ip <ID-instance> <floating-ip>
```

Jika `curl` ke floating IP gagal setelah VM DevStack dinyalakan ulang, pulihkan jaringan eksternal:

```bash
sudo ip addr add 172.24.4.1/24 dev br-ex
sudo ip link set br-ex up
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -s 172.24.4.0/24 -o ens33 -j MASQUERADE
```

Agar Windows dapat membuka floating IP (PowerShell Administrator):

```powershell
route add 172.24.4.0 mask 255.255.255.0 192.168.23.160
```

## Kendala utama

| Kendala | Solusi |
| --- | --- |
| Disk penuh saat `stack.sh` (root hanya 19 GB dari 40 GB) | `lvextend`, bersihkan dengan `unstack.sh` dan `clean.sh`, ulangi `stack.sh` |
| `br-ex` mati setelah VM dinyalakan ulang | Beri IP, aktifkan interface, `ip_forward`, dan NAT |
| nginx gagal terpasang di instance (DNS dan unduhan paket tersendat) | DNS diisi manual; website dilayani dengan Python `http.server` (halaman `website/index.html`) |

Rincian lengkap beserta screenshot ada di laporan.

## Catatan

- Password tidak disertakan di repository ini. Ganti `<password>` dan `<password-instance>` pada file di `config/` dengan password sendiri.
- Web server Python pada instance dijalankan dengan `sudo nohup python3 -m http.server 80 &` di folder `/var/www/html`, sehingga berhenti jika instance di-reboot. `config/web.yaml` menggunakan nginx; pada percobaan ini nginx gagal terpasang karena jaringan keluar instance yang tersendat.
- DevStack ditujukan untuk belajar dan pengujian, bukan untuk produksi.

## Sumber

- DevStack documentation: https://docs.openstack.org/devstack/latest/
