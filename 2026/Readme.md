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