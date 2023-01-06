# ansible_automation_EX457
This guide will walk through setting up GNS3 on and Ubuntu 20.04 host in GCP


### Create Ubuntu 20.04 host and enable nested virtualization on GCP
Create a boot disk from a public image
```
gcloud compute disks create nested-vm-disk --type=pd-standard --zone=northamerica-northeast1-c --image-family=ubuntu-2004-lts --image-project=ubuntu-os-cloud
```
Create a custom image from the boot disk
```
gcloud compute images create nested-ubuntu-2004 --source-disk nested-vm-disk --source-disk-zone northamerica-northeast1-c --licenses "https://www.googleapis.com/compute/v1/projects/vm-options/global/licenses/enable-vmx"
gcloud compute disks delete nested-vm-disk
```
Create a virtual machine in Google Compute Engine
```
gcloud compute instances create gns3host --zone northamerica-northeast1-c --min-cpu-platform "Intel Cascade Lake" --image nested-ubuntu-2004 --machine-type=n2-standard-2 --boot-disk-size=300GB --boot-disk-type=pd-standard
```
Open ports used to connect to GNS3
```
gcloud compute firewall-rules create gns3-inbound --description="open up the ports used by gns3" --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=tcp:3080,tcp:5000-10000 --source-ranges=0.0.0.0/0
```


### Install GNS3 on Ubuntu host
Update and upgrade all the installed packages
```
sudo apt update && sudo apt upgrade -y
```
Add gns3 repo ant install the server
```
sudo add-apt-repository ppa:gns3/ppa -y
```
Add IOU (Cisco IOS on Unix) support
```
sudo dpkg --add-architecture i386      
```                    
Remove docker if already installed
```
sudo apt remove docker docker.io -y
```
Add docker gpg key and repo
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
"deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) stable"
```
Install all required software
```
sudo apt update
sudo apt install gns3-server gns3-iou qemu-kvm docker-ce net-tools uml-utilities iptables-persistent -y
```
Check KVM
```
kvm-ok
```
Add gns3 user and to the required groups
```
sudo useradd gns3 --create-home
sudo passwd gns3
sudo usermod -aG ubridge,libvirt,kvm,docker gns3
```
Set up GNS3 service
```
sudo touch /etc/systemd/system/gns3.service
sudo vi /etc/systemd/system/gns3.service
```
Save this file
```
[Unit]
Description=GNS3 server
Wants=network-online.target
After=network.target network-online.target

[Service]
Type=forking
User=gns3
Group=gns3
PermissionsStartOnly=true
ExecStartPre=/bin/mkdir -p /var/log/gns3 /var/run/gns3
ExecStartPre=/bin/chown -R gns3:gns3 /var/log/gns3 /var/run/gns3
ExecStart=/usr/bin/gns3server --log /var/log/gns3/gns3.log \
 --pid /var/run/gns3/gns3.pid --daemon
ExecReload=/bin/kill -s HUP $MAINPID
Restart=on-abort
PIDFile=/var/run/gns3/gns3.pid

[Install]
WantedBy=multi-user.target
```
Set file permissions and configure GNS3 service to start on boot
```
sudo chown root /etc/systemd/system/gns3.service
sudo systemctl enable gns3.service
```
Set firewall rules to allow connection from remote dev environment
```
sudo ufw allow 3080/tcp
sudo ufw allow 5000:10000/tcp
```
Set up tap interface to allow communication from Ubuntu host to GNS3 objects via Cloud
```
sudo modprobe tun
sudo tunctl -t tap0
sudo ifconfig tap0 10.0.0.1 netmask 255.255.255.0 up
sudo ifconfig
```
Set NAT forwarding
```
sudo iptables -t nat -A POSTROUTING -o ens4 -j MASQUERADE
sudo iptables -A FORWARD -i tap0 -j ACCEPT
sudo iptables -t nat -vxnL
sudo tcpdump -i tap0  -s 1500
sudo tcpdump -i ens4  -s 1500 port not 22
```
Install Python 3.10, Pip and Ansible
```
sudo apt install software-properties-common -y
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt install python3.10
sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.8 1
sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.10 2
sudo update-alternatives --config python3
python3 --version
sudo apt install python3-pip
pip --version
pip install ansible paramiko ansible-pylibssh netaddr
ansible --version
```


### Configure GNS3
Now you can log into GNS3 and configure the installation
1. Go to Edit > Preferences and then click on Server
2. Make sure Enable local server is unchecked
3. Under Remote main server enter the ephemeral public IP address of you GCE instance
4. All other settings on this page can stay default
5. Click on IOS on Unix and enter the IOU license
6. Configure IOU device templates under IOU devices
7. Click OK
Next create a new project and add routers, a switch and a cloud
SSH to each router inside GNS3 to get SSH keys stored
```
ssh -o KexAlgorithms=+diffie-hellman-group-exchange-sha1 -c aes256-cbc developer@10.0.0.10
```


### Basic config for routers (developer/C1sco12345)
```
conf t
no service call-home 
no call-home
hostname R1
int e1/3
ip address 10.0.0.10 255.255.255.0
no shut
ip domain-name gns3.local
crypto key gen rsa mod 1024
username developer priv 15 secret C1sco12345
enable secret C1sco12345
line vty 0 4
login local
transport input ssh
exit
exit
wr mem
```

### Basic config for switches (developer/C1sco12345)
```
conf t
hostname S1
vlan 100
exit
int vlan 100
ip address 10.0.0.20 255.255.255.0
no shut
int range e1/0 - 3
switchport access vlan 100
ip domain-name gns3.local
crypto key gen rsa mod 1024
username developer priv 15 secret C1sco12345
enable secret C1sco12345
line vty 0 4
login local
transport input ssh
exit
exit
wr mem
```

### Basic config for vyos (vyos/vyos)
```
conf
set system host-name V1
set system domain-name gns3.local
set interfaces ethernet eth6 address 10.0.0.14/24
set interfaces ethernet eth0 address 10.1.3.2/30
set interfaces ethernet eth1 address 10.1.5.1/30
set service ssh port '22'
commit
save
```

### Basic config for arista (admin)
```
enable
zerotouch cancel
enable
conf t
hostname A1
int management 1
ip address 10.0.0.30/24
no shut
exit
ip route 0.0.0.0/0 10.0.0.1
wr
exit
```
