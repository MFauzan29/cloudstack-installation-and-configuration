# Apache Cloudstack Private Cloud Installation and Configuration

**Kelompok 6**
Anggota :
| Nama                          | NPM         | Github         |
|------------------------------|-------------|----------------|
| Dimas Dermawan               | 2206059654  | [@den-dimas](https://github.com/den-dimas)     |
| Irfan Yusuf Khaerullah       | 2206813290  | [@irfanyusuf13](https://github.com/irfanyusuf13)    |
| Aulia Anugrah Aziz           | 2206059364  | [@auli-aziz](https://github.com/auli-aziz) |
| Muhamad Fauzan               | 2206819054  | [@MFauzan29](https://github.com/MFauzan29) |
| Azzah Azkiyah Angeli Syahwa | 2106731390  | [@azzahangely](https://github.com/azzahangely) |


## Pendahuluan

### Apa itu Cloudstack

Cloudstack adalah software komputasi awan open-source yang dirancang untuk menerapkan dan mengelola jaringan mesin virtual (VM) dalam skala besar. Cloudstack merupakan platform Infrastructure-as-a-Service (IaaS) yang menyediakan serangkaian fitur lengkap untuk membangun dan mengelola lingkungan cloud yang dapat diskalakan. Platform ini mendukung berbagai jenis hypervisor, memiliki kemampuan API yang luas, dan menyediakan beragam alat bagi administrator cloud.

Dengan cloudstack, pengguna dapat mengelola cloud melalui antarmuka web, command line interface, dan RESTful API yang memiliki fitur lengkap. Cloudstack juga menyediakan API yang kompatibel dengan AWS EC2 dan S2 bagi organisasi yang ingin menerapkan cloud hybrid.

## Architecture Reference 
![messageImage_1747839942200](https://hackmd.io/_uploads/S1t82PiZgx.jpg)


## Environment Setup

### Spesifikasi Hardware

```
CPU : Intel Core i5 gen 8
RAM : 24 GB
Storage : 250GB
Network : Ethernet 100GB/s
Operating System : Ubuntu Server 24.04
```

### Network Address
```
Network Address : 10.10.0.0/16
Host IP address : 192.168.1.5 (KVM Host)
Gateway         : 192.168.1.101 (Virtual Router)
Management IP   : 192.168.1.5
System IP       : 10.10.10.1 (jaringan internal ke VM via VR)
Public IP       : 192.168.1.101 (VR interface ke luar)

```


## Configure Network

### Persiapan Sistem Operasi (Ubuntu Server)
#### Memastikan hostname sudah FQDN

```
hostname --fqdn
```

> Jika belum FQDN (misalnya: cloudstack1.localdomain), ubah /etc/hosts agar sesuai.

#### Install NTP (Chrony)
```
sudo apt update
sudo apt install chrony -y
sudo systemctl enable chrony
sudo systemctl start chrony
```

#### Konfigurasi Repository Cloudstack
Edit atau buat file:
```
sudo nano /etc/apt/sources.list.d/cloudstack.list
```
Tambahkan:
```
deb https://download.cloudstack.org/ubuntu focal 4.20
```
![image](https://hackmd.io/_uploads/SJnIBviWlx.png)

Tambahkan GPG key : 
```
wget -O - https://download.cloudstack.org/release.asc | sudo tee /etc/apt/trusted.gpg.d/cloudstack.asc
sudo apt update
```

### Instalasi Management Server
#### Install Cloudstack Management
```
sudo apt install cloudstack-management -y
```

### Instalasi dan Konfigurasi Database (MySQL)
#### Install MySQL Server
```
sudo apt install mysql-server
```
#### Konfigurasi MySQL
Edit file :
```
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```
Tambahkan dalam `mysqld`
```
server-id=source-01
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
max_connections=350
log-bin=mysql-bin
binlog-format = 'ROW'
```
![image](https://hackmd.io/_uploads/H1KjSPj-xl.png)

Restart MySQL:
```
sudo systemctl restart mysql

```
#### Amankan MySQL
```
sudo mysql_secure_installation
```

### Setup Database Untuk Cloudstack
#### Setup database menggunakan script CloudStack
```
cloudstack-setup-databases cloud:YourCloudDBPassword@localhost --deploy-as=root:YourMySQLRootPassword
```

Setelah sukses, akan muncul:
```
Successfully initialized the database.
```

#### Membuat Database & User untuk Cloudstack
Masuk ke MySQL sebagai root
```
sudo mysql -u root -p
```

Jalankan perintah berikut (ganti `your_secure_password`)

```
CREATE DATABASE cloud;
CREATE DATABASE cloud_usage;

CREATE USER 'cloud'@'localhost' IDENTIFIED BY 'your_secure_password';
CREATE USER 'cloud'@'%' IDENTIFIED BY 'your_secure_password';

GRANT ALL ON cloud.* TO 'cloud'@'localhost';
GRANT ALL ON cloud.* TO 'cloud'@'%';

GRANT ALL ON cloud_usage.* TO 'cloud'@'localhost';
GRANT ALL ON cloud_usage.* TO 'cloud'@'%';

GRANT PROCESS ON *.* TO 'cloud'@'localhost';
GRANT PROCESS ON *.* TO 'cloud'@'%';

FLUSH PRIVILEGES;
EXIT;
```

#### Jalankan Setup Database Cloudstack
```
sudo cloudstack-setup-databases cloud:your_secure_password@localhost \
  --deploy-as=root:<mysql_root_password> \
  -e file -m mymgmtkey -k mydbkey -i $(hostname -I | awk '{print $1}')
```

### Setup Management Server 
```
sudo cloudstack-setup-management
```
Jika berhasil akan muncul 
> “CloudStack Management Server setup is done.”

### Setup NFS Lokal 
#### Install NFS Server
```
sudo apt install nfs-kernel-server -y
sudo mkdir -p /export/primary /export/secondary
sudo chown -R nobody:nogroup /export
```
Edit file ekspor:
```
sudo nano /etc/exports
```
Isi : 
```
/export *(rw,async,no_root_squash,no_subtree_check)
```
![image](https://hackmd.io/_uploads/BJMeLvjWgg.png)

Aktifkan NFS : 
```
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

### Download System VM Template 
Contoh untuk KVM : 
```
sudo mkdir -p /mnt/secondary
sudo mount -t nfs localhost:/export/secondary /mnt/secondary

sudo /usr/share/cloudstack-common/scripts/storage/secondary/cloud-install-sys-tmplt \
  -m /mnt/secondary \
  -u http://download.cloudstack.org/systemvm/4.20/systemvmtemplate-4.20.0-x86_64-kvm.qcow2.bz2 \
  -h kvm -F
```

### Akses UI Cloudstack
![image](https://hackmd.io/_uploads/HJgdQwibgl.png)

#### Akses UI CloudStack
- URL: `http://<management-ip>:8080/client`
- Username: `admin`
- Password: `123`

## Konfigurasi KVM Hypervisor

KVM (Kernel-based Virtual Machine) adalah hypervisor tipe 1 yang terintegrasi langsung ke dalam kernel Linux. Dengan KVM, Linux berubah dari sistem operasi biasa menjadi platform virtualisasi penuh yang bisa menjalankan banyak sistem operasi (VM) secara bersamaan.

### Konfigurasi Agent
```
sudo apt install cloudstack-agent
```
**Jika menggunakan non root user tambahkan**
```
cloudstack ALL=NOPASSWD: /usr/bin/cloudstack-setup-agent
Defaults:cloudstack !requiretty
```
### Konfigurasi `libvirt`
``libvirt`` adalah library dan toolkit manajemen virtualisasi yang digunakan untuk mengelola hypervisor seperti KVM, QEMU, Xen, VMware, dan lainnya.

#### Edit file : 
```
sudo nano /etc/libvirt/libvirtd.conf
```
#### Set parameter berikut : 
```
listen_tls = 0
listen_tcp = 0
tls_port = "16514"
tcp_port = "16509"
auth_tcp = "none"
mdns_adv = 0
```
![image](https://hackmd.io/_uploads/ryrxBwiZeg.png)


Edit file : 
```
sudo nano /etc/default/libvirtd
```
Ubah : 
```
LIBVIRTD_ARGS="--listen"
```
Edit juga `/etc/libvirt/libvirt.conf`:
```
sudo nano /etc/libvirt/libvirt.conf
```
Tambahkan ini : 
```
remote_mode="legacy"
```
![image](https://hackmd.io/_uploads/BkUpNPoWxl.png)

Restart libvirt : 
```
sudo systemctl restart libvirtd
```

### Nonaktifkan AppArmor (Ubuntu)
```
sudo ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
sudo ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
sudo apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
sudo apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

### Konfigurasi Jaringan
#### Basic Networking (dengan Netplan)
Asumsikan satu interface fisik `eth0` dengan:
* VLAN 100 → Management (cloudbr0)
* VLAN 200 → Guest (cloudbr1)

```
sudo nano /etc/netplan/01-cloudstack.yaml
```
```
network:
  version: 2
  renderer: networkd

  ethernets:
    enp1s0:
      dhcp4: false
      dhcp6: false

  vlans:
    vlan200:
      id: 200
      link: enp1s0

  bridges:
    cloudbr0:
      interfaces: [enp1s0]
      addresses:
        - 192.168.1.5/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [1.1.1.1]
      parameters:
        stp: true
        forward-delay: 0

    cloudbr1:
      interfaces: [vlan200]
      dhcp4: false
      dhcp6: false
      parameters:
        stp: true
        forward-delay: 0
```
![image](https://hackmd.io/_uploads/BJfPUwsWxl.png)

Lalu aktifkan :
```
sudo netplan apply
```

```
sudo nano /etc/netplan/01-kvm-basic.yaml
```
Isi : 
```
network:
  version: 2
  ethernets:
    eth0: {}
  vlans:
    vlan100:
      id: 100
      link: eth0
    vlan200:
      id: 200
      link: eth0
  bridges:
    cloudbr0:
      interfaces: [vlan100]
      addresses: [192.168.42.11/24]
      gateway4: 192.168.42.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
      parameters:
        stp: true
    cloudbr1:
      interfaces: [vlan200]
      dhcp4: false
      parameters:
        stp: true
```

Lalu aktifkan : 
```
sudo netplan apply
```


#### Advanced Networking (dengan Netplan)
Asumsikan 2 interface:
* eth0 → cloudbr0 (management)
* eth1 → cloudbr1 (guest/public)
```
sudo nano /etc/netplan/01-kvm-advanced.yaml
```
Isi : 
```
network:
  version: 2
  ethernets:
    eth0: {}
    eth1: {}
  bridges:
    cloudbr0:
      interfaces: [eth0]
      addresses: [192.168.42.11/24]
      gateway4: 192.168.42.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
      parameters:
        stp: true
    cloudbr1:
      interfaces: [eth1]
      dhcp4: false
      parameters:
        stp: true
```
Aktifkan : 
```
sudo netplan apply
```
#### OpenVSwitch Mode
**Install OVS**
```
sudo apt install openvswitch-switch -y
```
**Setup OVS bridges dan VLAN**
```
sudo modprobe -r bridge
sudo ovs-vsctl add-br cloudbr
sudo ovs-vsctl add-port cloudbr eth0
sudo ovs-vsctl set port cloudbr trunks=100,200,300
sudo ovs-vsctl add-br mgmt0 cloudbr 100
sudo ovs-vsctl add-br cloudbr0 cloudbr 200
sudo ovs-vsctl add-br cloudbr1 cloudbr 300
```
**Setup Netplan**
```
sudo nano /etc/netplan/01-kvm-ovs.yaml
```
Isi : 
```
network:
  version: 2
  ethernets:
    eth0: {}
  bridges:
    mgmt0:
      interfaces: []
      addresses: [192.168.42.11/24]
      gateway4: 192.168.42.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
    cloudbr0:
      interfaces: []
      dhcp4: false
    cloudbr1:
      interfaces: []
      dhcp4: false
```
Aktifkan:
```
sudo netplan apply
```

### Konfigurasi Firewall
Buka port untuk KVM dan Cloudstack: 
```
sudo ufw allow 22
sudo ufw allow 1798
sudo ufw allow 16514
sudo ufw allow 5900:6100/tcp
sudo ufw allow 49152:49216/tcp
```
> Jika UFW aktif dan bridged network gagal, tambahkan di `/etc/ufw/before.rules` sebelum `COMMIT`:

### Tambahan Opsional
* Install `aria2` jika ingin mendukung Secondary Storage Bypass:
```
sudo apt install aria2 -y
```
* Install `ovmf` untuk support UEFI:
```
sudo apt install ovmf -y
```


### Konfigurasi Cloudstack

**Register Iso**
![image](https://hackmd.io/_uploads/rkf4OPoZee.png)

**Add Instances**
![image](https://hackmd.io/_uploads/rkwToDjZxl.png)

![image](https://hackmd.io/_uploads/HycknDobxl.png)

### Konfigurasi Web Server
* Update ubuntu
```
sudo apt update
```
* Install Apache2
```
sudo apt install apache2
```
* Memeriksa status aktif apache (pastikan Apache2 sudah aktif)
```
service apache2 status
```
* Mendownload uzip
```
sudo apt install unzip
```
* Mendownload template HTML
```
wget --content-disposition --trust-server-names "https://templatemo.com/download/templatemo_589_lugx_gaming"
```
* Melakukan unzip template HTML
```
unzip templatemo_589_lugx_gaming.zip -d lugx_gaming
```
* Rename default index.html
```
sudo mv /var/www/html/index.html /var/www/html/index_backup.html
```
* Copy template ke dalam folder /var/www/html
```
sudo cp -r ~/templatemo_589_lugx_gaming/* /var/www/html/
```
* Memberikan izin
```
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/
```
