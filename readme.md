# My Awesome Proxmox Setup

Juste some notes on setting up proxmox and stuff I modified to my own needs and like, also, some problems that I tweaked.

### download proxmox and burn with rufus (use dd option, otherwise you will get errors)
### note: Careful not to create a softraid on the physical raid.  when crating a raid; either you create it with the raid card (physical) or you create it with proxmox; otherwise if you loose a disk, you won't be able to replace it.

https://www.proxmox.com/en/downloads

## 1- resize disk to suit need:
     
     Problem I was having is proxmox not allowing me to use all space on disk. Which makes sense!  I mean, you don't want your vm's on the same disk as your  
     system, but in my case I wanted all the space to install my vm's.
     
     Solution:
     
     lvremove /dev/pve/data
     lvresize -l +100%FREE /dev/pve/root
     resize2fs /dev/mapper/pve-root

## 2- update, upgrades, etc

    mv /etc/apt/sources.list.d/pve-enterprise.list /etc/apt/sources.list.d/pve-enterprise.list.bak

    add these repo to /etc/apt/souces.list

    echo deb http://download.proxmox.com/debian/pve buster pve-no-subscription > /etc/apt/sources.list
   

    apt-get update && upgrade -yes
    apt-get dist-upgrade -yes

    apt-get autoclean 
    apt-get autoremove -yes

## 3- install some utilities:

    apt-get install htop tmux fail2ban samba speedtest-cli net-tools ntfs-3g ifupdown2 hdparm
    
## 4- change ssh port to 2299

    sed -i 's/#Port 22/Port 2299/' /etc/ssh/sshd_config
    sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
    systemctl restart ssh

    
## 5- create jail for ssh

     cp /etc/fail2ban/fail2ban.conf /etc/fail2ban/fail2ban.local
     cat << EOF > /etc/fail2ban/jail.local
     [DEFAULT]
     ignoreip = 127.0.0.1/8
     bantime  = 600
     findtime = 600
     maxretry = 3

     [sshd]
     enabled = true
     port = 2299
     EOF
     systemctl restart fail2ban

## 6- samba config

    nano /etc/samba/smb.conf

    #Just an example conf:
    
    [global]
    workgroup = WORKGROUP
    map to guest = Bad User
    log file = /var/log/samba/%m
    log level = 1
    
    [8TB]
    path = /mnt/8TB
    read only = no
    guest ok = yes

    systemctl restart smbd

## 7- update templates

    pveam update

## 8- backup grub

    rsync -avh --progress /boot/grub /xxx/xxx/xxx
    
## 9- GPU Passtrough

    nano /etc/default/grub

    GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt pcie_acs_override=downstream,multifunction nofb nomodeset video=vesafb:off,efifb:off"

    update-grub

    nano /etc/modules

    vfio
    vfio_iommu_type1
    vfio_pci
    vfio_virqfd

    echo "options vfio_iommu_type1 allow_unsafe_interrupts=1" > /etc/modprobe.d/iommu_unsafe_interrupts.conf
    echo "options kvm ignore_msrs=1" > /etc/modprobe.d/kvm.conf

    echo "blacklist radeon" >> /etc/modprobe.d/blacklist.conf
    echo "blacklist nouveau" >> /etc/modprobe.d/blacklist.conf
    echo "blacklist nvidia" >> /etc/modprobe.d/blacklist.conf
    echo "blacklist nvidiafb" >> /etc/modprobe.d/blacklist.conf

    lspci -v

    Look for your GPU and take note of the first set of numbers this is your PCI card address.

    Then run this command

    lspci -n -s (PCI card address)

    in my case:

    lspci -n -s 05:00

    This command gives use the GPU vendors number

    Use those numbers in this command (in my case:)

    echo "options vfio-pci ids=10de:06dd,10de:0be5 disable_vga=1"> /etc/modprobe.d/vfio.conf

    update-initramfs -u

    reboot

    Create Vm with:

    Bios is OMVF(UEFI)
    Machine is q35

## 10- bonding nics

    we need to create a bond0 and add all nics that we want to include than we add the bond to the linux bridge

   ![bond](https://user-images.githubusercontent.com/3466110/149634851-4a673e3d-e8df-4940-9cf4-bc5090c3f219.png)

## 11- installing megacli

    cd /opt
    apt install unzip alien libncurses5 wget
    wget https://docs.broadcom.com/docs-and-downloads/raid-controllers/raid-controllers-common-files/8-07-14_MegaCLI.zip
    unzip 8-07-14_MegaCLI.zip
    cd Linux 
    sudo alien MegaCli-8.07.14-1.noarch.rpm
    sudo dpkg -i megacli_8.07.14-2_all.deb
    /opt/MegaRAID/MegaCli/MegaCli64 -h

 ## 12- test I/O speed with hdparm

    
hdparm -tT --direct /dev/sda



cd /myFolderTheHardDisk/.
dd if=/dev/zero of=testfile bs=1M count=1024 conv=fdatasync,notrunc
 <br />


# Random note and cheats
 <br />


## Megacli Cheat

**check raid:**

 megacli -ldinfo -lALL -aALL

**check disk:**

 megacli -pdlist -aALL

**disk offline:**

megacli -pdoffline -physdrv[252:3] -a0

**mark as missing:**

megacli -pdmarkmissing -physdrv[252:3] -aAll


**mark as fail and prepare removal:**

megacli -pdprprmv -physdrv[252:3] -a0

**locate it:**

megacli -pdlocate -start -physdrv[252:3] -a0

**replace disk:**

megacli -PdReplaceMissing -PhysDrv[252:3] -Array0 -row0 -a0

**start rebuild:**

 megacli -PDRbld -Start -PhysDrv[252:3] -a0

**check progress rebuild:**

megacli -PDRbld -ShowProg -PhysDrv[32:2] -a0

**check batt:**

megacli -AdpBbuCmd -aAll

**write back:**

megali -LDSetProp WB -LALL -aALL

**read aahead:**

megacli -LDSetProp -RA -Immediate -LALL -aALL

**disk drive cache:**

megacli -LDSetProp -EnDskCache -Immediate -LAll 

source: https://cs.uwaterloo.ca/twiki/view/CF/MegaCli

## Observium Cheat

wget http://www.observium.org/observium_installscript.sh
chmod +x observium_installscript.sh
./observium_installscript.sh

## Listen to all interface

agentAddress udp:161

## Change "observium" to your preferred SNMP community string

com2sec readonly default observium

group MyROGroup v2c readonly

view all included .1 80

access MyROGroup "" any noauth exact all none none

## Update your location here

syslocation [40.705311,-74.2581883]
syslocation New York, United States
syscontact ITzGeek Admin <admin@itzgeek.local>

## Distro Detection

extend .1.3.6.1.4.1.2021.7890.1 distro /usr/bin/distro

# Useful VM

## PiHole

curl -sSL https://install.pi-hole.net | bash

## Ansible

apt install ansible

## Docker

curl -s https://gist.githubusercontent.com/mtnezm/502cdb812caa25a32ddd994f6fbff0df/raw/e7ec8f706fc88623f0fb097b4d9704c4e0b4bd9a/install_docker_debian.bash | sudo bash

## Portainer (docker manager)

docker volume create portainer_data

docker run -d -p 8000:8000 -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce

## nodered

sudo apt install build-essential

bash <(curl -sL https://raw.githubusercontent.com/node-red/linux-installers/master/deb/update-nodejs-and-nodered)


remove nag screen proxmox 8: 

sed -Ezi.bak "s/(Ext.Msg.show\(\{\s+title: gettext\('No valid sub)/void\(\{ \/\/\1/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js && systemctl restart pveproxy.service
