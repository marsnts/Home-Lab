# Home Lab Setup Guide
---
### What you need:

1. Hardware:
	1. Computer
	2. Network Switch
	3. Router

2. Software / Repositories / Packages:
	1. Proxmox VE Installer (iso image) via Rufus / Balena Etcher
	2. Ubuntu LTS Installer (iso image) via Rufus / Balena Etcher
	3. Git
	4. Nginx
	5. Postgres
	6. Python
	7. Django
	8. Gunicorn
	9. Wordpress
---
### System Diagram

![[diagram-export-7-15-2026-10_13_35-PM.png]]

---
### Installing Proxmox [To be done](https://medium.com/@upeka7/how-i-set-up-my-homeserver-part-2-installing-proxmox-ve-a76189c64984)




---
### Setting Up my Device

Currently there are no `sudo` package installed in Proxmox. In order to install sudo, logging in as `root` and installing `sudo` package is needed.

```bash
apt update && apt install sudo -y
```


Since I am using laptop as my Host PC, I disabled the hinge switch to access my device while laptop is close and to prohibit it to shutdown. It can be done using the following:
```bash
sudo nano /etc/systemd/logind.conf
```
Then edit the following lines to:
```
HandleLidSwitch=ignore
HandleLidSwitchDocked=ignore
LidSwitchIgnoreInhibited=yes
```
Save and exit.

Activate and apply changes using:
```bash
sudo systemctl restart systemd-logind.service
```

Closing the laptop lid would not make the system inaccessible and does not automatically sleep and shutdown.

I also wanted to have a secondary user to avoid using root as my main user when configuring the system itself. Creating user and adding them to the `sudo` group using the following commands:

```bash
adduser yourusername
usermod -aG sudo yourusername
```
###### Setting up WiFi in Proxmox

Ideally, Wi-Fi on Proxmox is not suitable for a normal Proxmox bridge configuration but there are some workarounds. I currently (at the time) only have 1 ethernet cable and does not have a 5-port switch (I plan to purchase one). I decided to temporarily connect it to Wi-Fi.

My default network config found on `/etc/network/interfaces` is:
```
auto vmbr0
iface vmbr0 inet static
	address 192.168.1.16/24
	gateway 192.168.1.1
	bridge-ports nic0
	bridge-stp off
	bridge-fd 0
	
iface nic1 inet manual
```

First, find the Wi-Fi interfaces:
```bash
ip link
```
or
```
ip a
```

It should look like this:
```bash
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: nic0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master vmbr0 state UP mode DEFAULT group default qlen 1000
    link/ether f4:39:09:77:6c:ea brd ff:ff:ff:ff:ff:ff
    altname enxf43909776cea
3: wlp3s0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether dc:a2:66:2b:ca:7f brd ff:ff:ff:ff:ff:ff
    altname wlxdca2662bca7f
4: vmbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether f4:39:09:77:6c:ea brd ff:ff:ff:ff:ff:ff
```

My Wi-Fi interface is `wlp3s0`

We need to install required packages first:
```bash
apt install wpasupplicant iw dnsmasq iptables -y
```

Then run:
```
iw dev
```
It should show:
```
root@node1:~# iw dev
phy#0
        Interface wlp3s0
                ifindex 3
                wdev 0x1
                addr dc:a2:66:2b:ca:7f
                type managed
                txpower 0.00 dBm
                multicast TXQ:
                        qsz-byt qsz-pkt flows   drops   marks   overlmt hashcol tx-bytes        tx-packets
                        0       0       0       0       0       0       0       0               0
root@node1:~#
```

###### Testing Wi-Fi connection
Before modifying Proxmox network bridge, testing if Wi-Fi interface can actually connect by creating a temporary configuration:

```bash
nano /etc/wpa_supplicant/wpa_supplicant-wlp3s0.conf
```

Paste this:
```
ctrl_interface=/run/wpa_supplicant
update_config=1
country=PH

network={
    ssid="YOUR_WIFI_NAME"
    psk="YOUR_WIFI_PASSWORD"
}
```

Protect the file:
```bash
chmod 600 /etc/wpa_supplicant/wpa_supplicant-wlp3s0.conf
```
Then test:
```bash
wpa_supplicant -B -i wlp3s0 -c /etc/wpa_supplicant/wpa_supplicant-wlp3s0.conf
```
Then check:
```bash
iw dev wlp3s0 link
```
It should display:
```
Connected to 30:40:74:ae:24:3c (on wlp3s0)
        SSID: Kullette_2.4G
```
Check:
```bash
ip route
```
and:
```bash
ip addr show wlp3s0
```

If it works, we need to change Wi-Fi config to permanent.

Create a backup of the current configuration:
```bash
cp /etc/network/interfaces /etc/network/interfaces.backup
```

Then edit:
```bash
nano /etc/network/interfaces
```
to:
```
auto lo
iface lo inet loopback

auto wlp3s0
iface wlp3s0 inet static
    address 192.168.1.16/24
    gateway 192.168.1.1
    wpa-conf /etc/wpa_supplicant/wpa_supplicant-wlp3s0.conf

auto vmbr0
iface vmbr0 inet static
    address 10.10.10.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0

iface nic0 inet manual
```

###### Enabling IP Forwarding within Proxmox

The Proxmox host needs to act as a router between VMs `10.10.10.0/24` and local are a network `192.168.1.0/24`.

Edit:
```
nano /etc/sysctl.conf
```
Add:
```bash
net.ipv4.ip_forward=1
```
Then apply:
```bash
sysctl -p
```
Then verify:
```bash
sysctl net.ipv4.ip_forward
```
It should show:
```bash
net.ipv4.ip_forward = 1
```

###### Configuring NAT

Anything coming from `10.10.10.0/24` network going out through Wi-Fi should be masqueraded as the Proxmox host.

```bash
iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o wlp3s0 -j MASQUERADE
```
Then allow forwarding:
```bash
iptables -A FORWARD -i vmbr0 -o wlp3s0 -s 10.10.10.0/24 -j ACCEPT
```
Allow forwarding traffic:
```bash
iptables -A FORWARD -i wlp3s0 -o vmbr0 -d 10.10.10.0/24 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

###### Give VMs DHCP

By using `dnsmasq` :

Edit:
```shell
nano /etc/dnsmasq.d/proxmox-vm.conf
```
Add:
```vim
interface=vmbr0
bind-interfaces

dhcp-range=10.10.10.100,10.10.10.200,255.255.255.0,12h

dhcp-option=3,10.10.10.1
dhcp-option=6,10.10.10.1

server=1.1.1.1
server=8.8.8.8
```
Then:
```shell
systemctl restart dnsmasq
```
Then:
```shell
systemctl enable dnsmasq
```

Then check everything:
```
iw dev
```
and:
```
iw dev wlp3s0 link
```

###### Changing Proxmox Host IP to Wi-Fi Interface

Edit:
```shell
nano /etc/network/interfaces
```
To:
```
auto lo
iface lo inet loopback

auto wlp3s0
iface wlp3s0 inet static
    address 192.168.1.69/24
    gateway 192.168.1.1
    wpa-conf /etc/wpa_supplicant/wpa_supplicant-wlp3s0.conf

auto vmbr0
iface vmbr0 inet static
    address 10.10.10.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0

iface nic0 inet manual
```

Apply configuration:
```shell
ifreload -a
```

You can now login to `192.168.1.69:8006` to access Proxmox Host.

The current System architecture would be:
```
                       Internet
                           │
                    ┌──────┴──────┐
                    │ Wi-Fi Router│
                    │ 192.168.1.1 │
                    └──────┬──────┘
                           │
                      Wi-Fi (WPA)
                           │
	                     wlp3s0
	                  192.168.1.69/24
                           │
                  ┌────────┴────────┐
                  │     Proxmox     │
                  │      Host       │
                  │                 │
                  │ IP forwarding   │
                  │ NAT / iptables  │
                  └────────┬────────┘
                           │
                        vmbr0
                      10.10.10.1/24
                           │
                ┌──────────┼──────────┐
                │          │          │
              VM 101     VM 102     VM 103
            10.10.10.x  10.10.10.x  10.10.10.x
                │          │          │
                └──────────┴──────────┘
                           │
                        dnsmasq
                       DHCP + DNS
```

Note that `192.168.1.0/24` is the physical/home network and `10.10.10.0/24` is the VM/private network.

---
### Installing Ubuntu

##### Creating a USB Flashdrive Installer (if you want a standalone system for a server)

You can install the Ubuntu Server 26.04 LTS image file using this [link](https://ubuntu.com/download/server) and use [rufus](https://rufus.ie/en/) and [balenaEtcher](https://etcher.balena.io/) to flash the image file into the USB flash drive for installation.

In my case, I used Rufus
![[Pasted image 20260716152118.png|313]]

Enter the BIOS/UEFI on your computer while the USB Flash drive with the installer plugged in. Boot from the USB Flash drive and install Ubuntu server to your liking but I would personally enable OpenSSH.

##### Server Setup
###### Root Access

It is bad to log in as root. Applications are meant to be run with non-administrative security. Logging in directly as root removes all system safety rules. The root user has unlimited access to change, delete, or break anything on the systems. Thus creating a user with ``sudo`` privileges would make the CLI experience a bit more comfortable

###### Creating User with *sudo* Privileges

*sudo* stands for "**Superuser Do**", it lets you perform high-level tasks like installing software, repositories, packages, or changing system files without logging into the main administrator (root) account. ``sudo`` privileges allow a regular user to temporarily run computer commands with administrator (root) rights. 

Use the ``adduser`` command to add a new user to your system.
*Note: replace the `<name>` with any name you want*
```bash
adduser <name> 
```

You will be prompted to create and verify a password for the user.

You will also be asked to fill some information about the user. It is optional, you can leave them blank.

Use the `usermod` command to add the user into the *sudo* group.
```bash
usermod -aG sudo <name>
```

Now you can log into your server remotely within your local network if you enabled OpenSSH.
```bash
ssh <name>@<IP Address>
```

Although every time you run `sudo` commands, you will be asked for the user password. If you wanted to avoid being asked for the password every time you run `sudo` commands, we can edit the root configuration file.
```bash
sudo visudo
```
or
```bash
sudo nano /etc/sudoers
```
And add the following line to the sudoers list.
```bash
<name> ALL = NOPASSWD : ALL
```
This means the user `<name>` can skip the password check for all tasks.

###### Logging in with SSH-Key

If you currently do not have an SSH Key on your computer, you can generate it with the keygen command.
```bash
ssh-keygen
```

You will be asked if you wanted to choose a password for your SSH key. This is optional and you can leave it blank.

At the end, copy your public key to your server:
```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub <name>@<IP address>
```
*Note: `id_ed25519` and `id_rsa` is a Private key; you should not share this file. While `id_ed25519.pub` and `id_rsa.pub` is a Public Key; it is safe to share. `id_rsa` can be encountered in older machines while `id_ed25519` can be found in modern machines.*

Then enter your password. Then you can always login to your server without password.

###### Disabling root login

Disabling direct root login prohibits hackers or non-authorized personnel from accessing the root. It forces individuals to log in with a normal user and use `sudo` to do administrative tasks. We can disable root login thru ssh by editing `sshd_config` file.

```bash
sudo nano /etc/ssh/sshd_config
```

Review the file and look for the `PermitRootLogin` line.
```bash
PermitRootLogin yes
```
change `yes` to `no`
```bash
PermitRootLogin no
```
Save and close the file and restart sshd daemon to read the modified configuration.
```bash
sudo systemctl restart sshd
```

###### Firewall

Firewalld is installed by default in Ubuntu 26.04 LTS but you can always manually install firewalld.
```bash
sudo apt install firewalld
```
Then enable firewall services and it will automatically start at boot.
```bash
sudo systemctl enable firewalld
```
Verify if firewall service is running and reachable.
```bash
sudo firewall-cmd --state
```
It should display:
```bash
running
```

To allow traffic of HTTP and HTTPS for interfaces in the "public" zone.
```bash
sudo firewall-cmd --zone=public --permanent --add-service=http
sudo firewall-cmd --zone=public --permanent --add-service=https
sudo firewall-cmd --reload
```
Verify if HTTP and HTTPS traffic is allowed.
```bash
sudo firewall-cmd --zone=public --permanent --list-services
```
It should display:
```bash
dhcpv6-client http https ssh
```

###### Nginx

Install nginx by typing the following command:
```bash
sudo apt install nginx
```

Then enable the nginx service so it will start up automatically at boot:

```bash
sudo systemctl enable nginx
```

###### Git

Installing git allows you to have version control over your system. You can install Git using the following command:
```bash
sudo add-apt-repository ppa:git-core/ppa
sudo apt update
sudo apt install git
```

Setting your Git identity ensures identification who made changes to your repositories. You can configure your git identity using the following command:
```bash
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
```

Confirm the set identity using the following:
```bash
git config --global --list
```

You should see `use.name` and `user.email` with the information you just entered.




###### GitHub

You should have an account at [github.com](https://github.com). Sign in if you do not have an account. 

Create a new repository at the upper right section at GitHub:
![[Pasted image 20260806081840.png]]

Enter details:
![[Pasted image 20260806082015.png]]

At your Ubuntu system, run these commands to connect your local repository to GitHub. You should have set up your repository with Git beforehand.
```bash
git remote add origin https://github.com/your_GitHub_username/project_name
git branch -M main
git push -u origin main
```

`git remote add origin` connects local Git where your remote repository is located.
`origin` is the standard label for primary remote
`git branch -M main` renames your current branch to main if isn't already.
`git push -u origin main` uploads your entire commit history from local Git to GitHub.
`-u` flags your local branch to the remote so future pushes are simpler
	Note: you can replace `main` in `git push -u origin main` to push other branches
###### Docker (WIP)

Installing Docker allows containerization over your developed apps.

[https://docs.docker.com/engine/install/ubuntu/](https://docs.docker.com/engine/install/ubuntu/)

To fix docker permissions, use the following command:
```bash
sudo groupadd -f docker
sudo usermod -aG docker $USER
newgrp docker
```
This creates a docker group within the system and add active user to the docker group.

###### Java (WIP)

[https://sdkman.io/install](https://sdkman.io/install)

###### Node.JS (WIP)

[https://tecadmin.net/how-to-install-nvm-on-ubuntu-20-04/](https://tecadmin.net/how-to-install-nvm-on-ubuntu-20-04/)

---

### Setting up a Project / Repository

###### Creating a project / repository

Every project lives inside a folder. Creating a dedicated folder for your projects and initializing Git so that Git can track everything inside it.

Move the path to your repository using:
```bash
cd ~/Desktop
```

Create your project folder using:
```bash
mkdir project_folder
cd project_folder
```

Create a markdown file to document your project:
```bash
sudo nano readme.md
```
Save then exit

Initialize Git within your repository using:
```bash
git init
git add readme.md
git commit -m "Add initial commit"
```

You can verify if your commit was saved:
```bash
git log
```

Every time you made changes and wanted to save it to your Git, you should always commit your changes.

To check if you have uncommited changes, you can enter the command:
```bash
git status
```
It should show `nothing to commit, working tree clean` , if every changes is committed. 

---
### Setting up Nginx

