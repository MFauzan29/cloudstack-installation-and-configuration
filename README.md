# Apache Cloudstack Private Cloud Installation and Configuration

Contributors :
| Nama                          | NPM         | Github         |
|------------------------------|-------------|----------------|
| Dimas Dermawan               | 2206059654  | ---     |
| Irfan Yusuf Khaerullah       | 2206813290  | [@irfanyusuf13](https://github.com/irfanyusuf13) |
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
Operating System : Ubuntu Server 22.04
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

### Edit the network configuration file within /netplan directory

```
cd /etc/netplan
sudo nano ./0*.yaml
```


### Update System Packages 

```
sudo apt update && sudo apt upgrade -y
```

### Install Required Packages
```
sudo apt install openjdk-11-jdk mariadb-server qemu-kvm libvirt-daemon-system \
libvirt-clients bridge-utils nfs-common python3-pip uuid-runtime -y
```

#### Issue :  
```
sudo apt clean  , sudo apt update , sudo apt install -y --fix-missing ovmf
```

### Configure MariaDB

```
sudo mysql_secure_installation

sudo mysql -u root -p

CREATE DATABASE cloud;
CREATE USER 'cloud'@'%' IDENTIFIED BY 'cloud-password';
GRANT ALL PRIVILEGES ON cloud.* TO 'cloud'@'%';
FLUSH PRIVILEGES;
EXIT;

```

