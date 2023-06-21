## ESXi-Networking-Project

Virtual networking using ESXi with routing, DNS, DHCP, E-mail servers.

## Network topology

## ESXi network preparation

Create port 3 groups; pg_outside, pg_dmz, pg_inside
Create 2 virtual switches, one for inside and the other for the DMZ.

## VM Specifications

### VMs

| VM       | HDD  | RAM   | OS                 |
| ---------| ---- | ----- | ------------------ |
| RTR      | 4GB  | 768M  | Debian 10(64bits)  |
| DMZ      | 4GB  | 768M  | Debian 10(64bits)  | 
| Failover | 4GB  | 512M  | Debian 10(64bits)  |
| Win10    | 30GB | 3GB   | Windows 10(64bits) |

*All debian machines have 1 CPU.*
*Thin provisioned on HDD is recommended*

### Network interfaces

#### RTR
| Interface | IP Address     | Portgroup |
| --------- | -------------- | --------- |
| ens192    | pg_outside     |
| ens224    | pg_inside      |
| ens256    | pg_dmz         | 

#### DMZ

| Interface | IP Address | Portgroup | Gateway |
| -- | -- | -- | -- |
| ens192 | DHCP | | 172.31.0.1 |

#### Failover

| Interface | IP Address | Portgroup | Gateway
| -- | -- | -- | -- |
| ens192 | 172.31.0.2 | pg_inside |172.31.0.1

#### Window
| Interface | IP Address |
| -- | -- |
| ens192 | DHCP |

## Debian installation
 - Boot your Debian ISO
 - Select "Install using graphical interface"
 - Select your location and keyboard layout
 - Debian will attempt to configure its network through DHCP, but it will fail, ignore this and select *Continue*
![enter image description here](https://i.imgur.com/2IgLQdV.png)
 - Select *Configure network manually*
 - Input the correct IP Address and CIDR mask according to the table below
 - Input the correct gateway
 - It will ask to input name servers(DNS), you can leave this field blank or input 
 - Change the hostname
 - Input a domain name
 - Then it will ask for root password and to create a new user.
 - Adjust the clock/timezone
 - Select *Guide - user entire disk and set up LVM*
 - Separate /home partition
 - Accept the current partitioning scheme
 - Use 3.8GB as for the amount of volume
 - Confirm partition settings

![enter image description here](https://i.imgur.com/tAgQv1V.png)
- Select *Yes*
- Now, Debian will finish installation, this might take some time.
- Install GRUB boot loader as primary drive


## RTR Configuration

Connect to your RTR machine though SSH

### Networking

Before configuring anything, make sure you go in SU.

    su -

Check your interfaces and their MAC addresses

    ip a

Double check their MACs in  ESXi and use the correct IP address.
	
	# DMZ
    allow-hotplug ens224
	iface ens224 inet static
        address 172.31.0.1/24
    # Inside
    allow-hotplug ens224
	iface ens224 inet static
        address 172.31.0.1/24

After adding these lines, we need to restart.

    systemctl restart networking && systemctl reboot

Test connectivity by pinging each host.

### Enable IP forwarding

    nano /etc/sysctl.conf
	net.ipv4.ip_foward=1
	sysctl -p

#### DHCP

    apt install -y isc-dhcp-server
    nano /etc/dhcp/dpcpd.conf

Change the domain name to what you chose.

    option domain-name ""

Put the IP of each interface

    option domain-name-servers 192.168.15.x, 192.168.31.1, 172.31.0.1;

Add this

    key DHCP_UPDATER {
		algorithm HMAC-MD5.SIG-ALG.REG.INT;
		secret pRP5FapFoJ95JEL06sv4PQ==;
	};

Now, add all subnets to the configuration

    subnet 192.168.31.0 netmask 255.255.255.0 {
		range 192.168.31.128 192.168.31.191;
		option routers 192.168.31.1;
		option broadcast-address 192.168.31.255;
	}

Do this for 172.31.0.0/24

If needed, you can add these lines to make a host always have the same IP.

    host 172.31.0.100 {
	  hardware ethernet 00:50:56:86:7c:cf;
	  fixed-address 172.31.0.100;
	  option routers 172.31.0.1;
	  option broadcast-address 172.31.0.255;
	}

Got to /etc/default/ and open isc-dhcp-server
Add your interfaces to this list

    INTERFACESv4="ens192 ens224 ens256"

This tells DHCP server to listen on these interfaces

Restart DHCP server.

If there was no errors, you can go to the DMZ/failover and test it by editing /etc/network/interfaces. 
Replace **static** to **dhcp** and commenting all other configurations.
Restart networking services and run the command **dhclient -r** and **dhclient -v**

![enter image description here](https://i.imgur.com/E8BZ3Ft.png)

#### Bind9 (DNS)

#### Routing (IPTables)

## DMZ Configuration

## Windows configuration

## HTTP/HTTPS web server

## Easy-RSA - CA & Certificates creation
