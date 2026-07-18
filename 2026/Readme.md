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
``` 
adduser <name> 
```

You will be prompted to create and verify a password for the user.

You will also be asked to fill some information about the user. It is optional, you can leave them blank.

Use the `usermod` command to add the user into the *sudo* group.
```
usermod -aG sudo <name>
```

Now you can log into your server remotely within your local network if you enabled OpenSSH.
```
ssh <name>@<IP Address>
```

Although every time you run `sudo` commands, you will be asked for the user password. If you wanted to avoid being asked for the password every time you run `sudo` commands, we can edit the root configuration file.
```
sudo visudo
```
or
```
sudo nano /etc/sudoers
```
And add the following line to the sudoers list.
```
<name> ALL = NOPASSWD : ALL
```
This means the user `<name>` can skip the password check for all tasks.

###### Logging in with SSH-Key

If you currently do not have an SSH Key on your computer, you can generate it with the keygen command.
```
ssh-keygen
```

You will be asked if you wanted to choose a password for your SSH key. This is optional and you can leave it blank.

At the end, copy your public key to your server:
```
ssh-copy-id -i ~/.ssh/id_rsa.pub <name>@<IP address>
```
*Note: `id_ed25519` and `id_rsa` is a Private key; you should not share this file. While `id_ed25519.pub` and `id_rsa.pub` is a Public Key; it is safe to share. `id_rsa` can be encountered in older machines while `id_ed25519` can be found in modern machines.*

Then enter your password. Then you can always login to your server without password.

###### Disabling root login

Disabling direct root login prohibits hackers or non-authorized personnel from accessing the root. It forces individuals to log in with a normal user and use `sudo` to do administrative tasks. We can disable root login thru ssh by editing `sshd_config` file.

```
sudo nano /etc/ssh/sshd_config
```

Review the file and look for the `PermitRootLogin` line.
```
PermitRootLogin yes
```
change `yes` to `no`
```
PermitRootLogin no
```
Save and close the file and restart sshd daemon to read the modified configuration.
```
sudo systemctl restart sshd
```

###### Firewall

Firewalld is installed by default in Ubuntu 26.04 LTS but you can always manually install firewalld.
```
sudo apt install firewalld
```
Then enable firewall services and it will automatically start at boot.
```
sudo systemctl enable firewalld
```
Verify if firewall service is running and reachable.
```
sudo firewall-cmd --state
```
It should display:
```
running
```

To allow traffic of HTTP and HTTPS for interfaces in the "public" zone.
```
sudo firewall-cmd --zone=public --permanent --add-service=http
sudo firewall-cmd --zone=public --permanent --add-service=https
sudo firewall-cmd --reload
```
Verify if HTTP and HTTPS traffic is allowed.
```
sudo firewall-cmd --zone=public --permanent --list-services
```
It should display:
```
dhcpv6-client http https ssh
```
