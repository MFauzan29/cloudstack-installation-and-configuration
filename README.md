# Apache Cloudstack Private Cloud Installation and Configuration

Contributors :
| Nama                          | NPM         | Github         |
|------------------------------|-------------|----------------|
| Dimas Dermawan               | 2206059654  | ---     |
| Irfan Yusuf Khaerullah       | 2206813290  | [@irfanyusuf13](https://github.com/irfanyusuf13)    |
| Aulia Anugrah Aziz           | 2206059364  | [@auli-aziz](https://github.com/auli-aziz) |
| Muhamad Fauzan               | 2206819054  | [@MFauzan29](https://github.com/MFauzan29) |
| Azzah Azkiyah Angeli Syahwa | 2106731390  | [@azzahangely](https://github.com/azzahangely) |


## Pendahuluan

### Apa itu Cloudstack



## Environment Setup

### Spesifikasi Hardware

```
CPU : Intel Core i5 gen 8
RAM : 24 GB
Storage : 
Network : Ethernet 100GB/s
Operating System : Ubuntu Server 24.04
```

### Network Address
```
Network Address : 
Host IP address :
Gateway : 
Management IP :
System IP :
Public IP :
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
