
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
 - Input the correct IP Address and CIDR mask according to the table above
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

    option domain-name-servers x.x.x.x;

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

    host x.x.x.x {
	  hardware ethernet xx:xx:xx:xx:xx:xx;
	  fixed-address x.x.x.x;
	  option routers x.x.x.x;
	  option broadcast-address x.x.x.x;
	}

Go to /etc/default/ and open isc-dhcp-server
Add your interfaces to this list

    INTERFACESv4="ens192 ens224 ens256"

This tells DHCP server to listen on these interfaces

Restart DHCP server.

    systemctl restart isc-dhcp-server

If there was no errors, you can go to the DMZ/failover and test it by editing /etc/network/interfaces. 
Replace **static** to **dhcp** and commenting all other configurations.
Restart networking services and run the command **dhclient -r** and **dhclient -v**

![enter image description here](https://i.imgur.com/E8BZ3Ft.png)

#### Bind9 (DNS)

##### Installation
    apt install bind9 bind9-dnsutils bind9-doc bind9-utils bind9utils
    cd /etc/bind

Create a db by copying one of the existing ones inside the folder

##### Configuration

named.conf.local

    //
	// Do any local configuration here
	//
	
	// Consider adding the 1918 zones here, if they are not used in your
	// organization
	//include "/etc/bind/zones.rfc1918";

	key DHCP_UPDATER {
	         algorithm HMAC-MD5.SIG-ALG.REG.INT;
	         secret pRP5FapFoJ95JEL06sv4PQ==;
	};

	zone "zone.example" {
	        type master;
	        file "/var/cache/bind/db.zone.example";
	        allow-update { key DHCP_UPDATER; };
	};

	zone "31.172.in-addr.arpa" {
	        type master;
	        file "/var/cache/bind/db.172.31.zone.example";
	        allow-update { key DHCP_UPDATER; };
	};

named.conf.options

    options {
        directory "/var/cache/bind";

        // If there is a firewall between you and nameservers you want
        // to talk to, you may need to fix the firewall to allow multiple
        // ports to talk.  See http://www.kb.cert.org/vuls/id/800113

        // If your ISP provided one or more IP addresses for stable
        // nameservers, you probably want to use them as forwarders.
        // Uncomment the following block, and insert the addresses replacing
        // the all-0's placeholder.

        forwarders {
         8.8.8.8;
        };

        //========================================================================
        // If BIND logs error messages about the root key being expired,
        // you will need to update your keys.  See https://www.isc.org/bind-keys
        //========================================================================
        dnssec-validation auto;
        auth-nxdomain no;
        allow-recursion { any; };
        listen-on-v6 { any; };
};

#### Routing (IPTables)

    apt install iptables netfilter-persistent iptables-persistent

Enable NAT

    sudo iptables -t nat -A POSTROUTING -o ens192 -j MASQUERADE

Where **ens192** is the outside interface.

    netfilter-persistent reload

## DMZ Configuration

### Apache2 - HTTP/HTTPS web server

    apt install apache2

Enable HTTPS

    a2ensite default-ssl.conf 

### Easy-RSA - CA & Certificates creation

    apt install easy-rsa
    cd /etc
    cp -R /usr/share/easy-rsa/ .
    cd easy-rsa/
    cp vars.example vars
    
    ./easyrsa init-pki
	./easyrsa build-ca nopass
	
	./easyrsa --subject-alt-name="DNS:www.example.org" gen-req www.example.org nopass
	./easyrsa sign-req server www.example.org

	./easyrsa --subject-alt-name="DNS:smtp.example.org" gen-req smtp.example.org nopass
	./easyrsa sign-req server smtp.example.org

	./easyrsa --subject-alt-name="DNS:pop.example.org" gen-req pop.example.org nopass
	./easyrsa sign-req server pop.example.org

## Email server

### Installation

    apt-get install exim4-daemon-heavy sasl2-bin dovecot-pop3d dovecot-imapd
    cd /etc
	adduser Debian-exim ssl-cert
	adduser Debian-exim sasl
	adduser dovecot ssl-cert

    dpkg-reconfigure exim4-config

### Mail Server configuration
 - Select *internet site; mail is sent and received directly using SMTP*
 - Enter mail name (*example.org*)
 - *IP-addresses to listen for incoming SMTP connections:* Leave it blank
 - Leave the relay mail blank
 - Leave the machines relay blank
 - Select *No*
 - Select *Maildir format in home directory*
 - Select *No*
 
 **nano /etc/default/saslauthd**
    
		START=yes

**nano /etc/dovecot/conf.d/10-ssl.conf**
Update **ssl_cert** and **ssl_key**
After that, change the permissions of the certificate and key.
Example:

    chmod 777 /etc/ssl/private/*
	chmod 777 /etc/ssl/certs/*
	chmod 777 /etc/ssl/private
	chmod 777 /etc/ssl/certs/
	chmod 777 /etc/ssl/certs

**nano /etc/exim4/exim4.conf.template**

    MAIN_TLS_ENABLE=yes
	MAIN_TLS_CERTIFICATE=/etc/ssl/certs/.crt
	MAIN_TLS_PRIVATEKEY=/etc/ssl/private/.key

Uncomment these:

     plain_saslauthd_server:
	   driver = plaintext
	   public_name = PLAIN
	   server_condition = ${if saslauthd{{$auth2}{$auth3}}{1}{0}}
	   server_set_id = $auth2
	   server_prompts = :
	   .ifndef AUTH_SERVER_ALLOW_NOTLS_PASSWORDS
	   server_advertise_condition = ${if eq{$tls_in_cipher}{}{}{*}}
	   .endif

Run this command:

    echo "tls_on_connect_ports = 465" > /etc/exim4/exim4.conf.localmacros

**nano /etc/default/exim4**

    SMTPLISTENEROPTIONS='-oX 25:465:587:10025 -oP /run/exim4/exim.pid'

Start exim

    cd /etc/skel
    
	mkdir Maildir && cd Maildir && maildirmake.dovecot .
	systemctl enable saslauthd exim4 dovecot
	systemctl restart saslauthd exim4 dovecot
	systemctl status saslauthd exim4 dove

## Windows configuration

To make your browser from warning the user that the HTTPS server is insecure, you can upload your ca.crt to your Windows machine and install it.

Get DHCP lease:

    ipconfig /release
    ipconfig /renew

## DHCP Fail-over

