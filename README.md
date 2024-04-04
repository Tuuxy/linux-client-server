A solo project to learn Linux ( see Assignment.md for references )

For both the Linux server and the workstation I chose Ubuntu because it's a stable distribution and since it's user friendly it'll be easier for employees that are used to windows. I chose the same distribution for the server and the workstation for simplicity and compatibility.

After the server is installed I need to set a root password and add my user to the sudoers:


```
sudo su
passwd 

usermod -aG sudo karys
exit

sudo apt update
```

Then I configured ssh so I could connect to the server from my machine : 

```
sudo apt install openssh-server

sudo systemctl start ssh
sudo systemctl enable ssh

```

I went to /etc/ssh/sshd_config to change : 

```
#PasswordAuthentication yes
to 
PasswordAuthentication no

then :
sudo systemctl restart ssh

```

Now i can connect to the server by ssh.

I need to setup the server to use a static IP address : 

```
# create the netplan config file , and delete the existing ones
sudo nano /etc/netplan/01-static.yaml

network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      dhcp4: no
      addresses: [192.168.124.10/24]  
      gateway4: 192.168.124.1         
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4] 

sudo chmod 600 /etc/netplan/01-static.yaml
sudo netplan apply

now ip a gives us the ip : 192.168.124.10 as configured

```

Now to configure the dhcp server : 

```
sudo apt install isc-dhcp-server

sudo nano /etc/default/isc-dhcp-server
=> replace the last 2 lines : 

INTERFACESv4="enp1s0"
#INTERFACESv6=""

sudo nano /etc/dhcp/dhcpd.conf
=> and add those lines : 

```

```
default-lease-time 600;
max-lease-time 7200;

ddns-update-style none;

authoritative;

option subnet-mask 255.255.255.0;
option broadcast-address 192.168.124.255;                                      
option domain-name-servers 192.168.124.10;
option domain-name "karys.becode";                                
option routers 192.168.124.1;                                        

subnet 192.168.124.0 netmask 255.255.255.0 {
        range 192.168.124.100 192.168.124.120;                            
}

```

Then restart the dhcp server : 

```
sudo systemctl restart isc-dhcp-server
```

On the workstation you can go to the terminal : 

```

sudo dhclient -r
sudo dhclient -v 

```

![Workstation DHCP Verification](/Assets/dhcp-verification-workstation.png)

And on the server : 

```
sudo nano /var/lib/dhcp/dhcpd.leases

```

![Server DHCP Verification](/Assets/dhcp-verification-server.png)

To install the DNS server using bind ( berkeley internet name domain ):

```
sudo apt install bind9

sudo nano /etc/bind/named.conf.options

options {
    directory "/var/cache/bind";

    // If there is a firewall between you and nameservers you want
    // to talk to, you may need to fix the firewall to allow multiple
    // ports to talk.  See http://www.kb.cert.org/vuls/id/800113

    // If your ISP provided one or more IP addresses for stable 
    // nameservers, you probably want to use them as forwarders.  
    // Uncomment the following block, and insert the addresses replacing 
    // the all-0's placeholder.

    // forwarders {
    //      8.8.8.8;
    //      8.8.4.4;
    // };

    //========================================================================
    // If BIND logs error messages about the root key being expired,
    // you will need to update your keys.  See https://www.isc.org/bind-keys
    //========================================================================
    dnssec-validation auto;

    auth-nxdomain no;    
    listen-on-v6 { none; };
};

# Then configure the zones : 

sudo nano /etc/bind/db.karys.becode

;
; BIND data file for karys.becode
;
$TTL    604800
@       IN      SOA     ns.karys.becode. admin.karys.becode. (
                                3         ; Serial
                           604800         ; Refresh
                            86400         ; Retry
                          2419200         ; Expire
                           604800 )       ; Negative Cache TTL
;
@       IN      NS      ns.karys.becode.
@       IN      A       192.168.124.10
ns      IN      A       192.168.124.10

# Configure Reverse Zone 

sudo nano /etc/bind/db.192

;
; BIND reverse data file for local loopback interface
;
$TTL    604800
@       IN      SOA     ns.karys.becode. admin.karys.becode. (
                                2         ; Serial
                           604800         ; Refresh
                            86400         ; Retry
                          2419200         ; Expire
                           604800 )       ; Negative Cache TTL
;
@       IN      NS      ns.karys.becode.
10      IN      PTR     ns.karys.becode.

# Update bind configuration

sudo nano /etc/bind/named.conf.local

  GNU nano 6.2                                                                                  /etc/bind/named.conf.local *                                                                                         
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";

zone "karys.becode" {
        type master;
        file "/etc/bind/zones/karys.becode.zone";
};

zone "124.168.192.in-addr-arpa" {
        type master;
        file "/etc/bind/db.192"
};

# Then restart BIND

sudo systemctl restart bind9

sudo reboot
```

Then on the workstation VM : 

```

sudo nano /etc/systemd/resolved.conf

# Un-comment the DNS= line 
DNS=192.168.124.10

sudo systemctl restart systemd-resolved

sudo nano /etc/resolv.conf

# Check nameserver IP

nameserver 192.168.124.10

```

Then you can verify that your DNS server works on your workstation : 

![DNS Verification](/Assets/dns-verification.png)

![DNS karys.becode](/Assets/dns-karys.becode.png)


Setting up apache :

```
sudo apt install apache2
sudo systemctl start apache2
sudo systemctl enable apache2

```

Setting up mariadb : 

```
sudo apt install mariadb-server

# This script will secure your mariadb installation
sudo mysql_secure_installation

```

Setting up the GLPI : 

Download GLPI. Then, extract the downloaded file and move it to your web server's document root.

```
wget https://github.com/glpi-project/glpi/releases/download/10.0.0-rc1/glpi-10.0.0-rc1.tgz -O glpi.tar.gz
tar -xzvf glpi.tar.gz
sudo mv glpi /var/www/html/

sudo chown -R www-data:www-data /var/www/html/glpi
sudo chmod -R 755 /var/www/html/glpi

```

Configure apache for glpi

```
sudo nano /etc/apache2/sites-available/glpi.conf

# Add the following : 

<VirtualHost *:80>
    ServerAdmin admin@karys.becode
    DocumentRoot /var/www/html/glpi
    ServerName karys.becode

    <Directory /var/www/html/glpi/>
        Options FollowSymlinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/glpi_error.log
    CustomLog ${APACHE_LOG_DIR}/glpi_access.log combined
</VirtualHost>


# Then enable the virtual host

sudo a2ensite glpi.conf
sudo systemctl reload apache2

```

Create a user and a database for your mariadb and give it the permissions to access it :

```
sudo mariadb -u root -p

CREATE DATABASE <database_name>;
CREATE USER '<username>'@'localhost' IDENTIFIED BY '<password>';
GRANT ALL PRIVILEGES ON <database_name>.* TO '<username>'@'localhost';
FLUSH PRIVILEGES;
EXIT;

```

After that you can go to your webpage : 

http://ip/glpi ( for me it was http://192.168.124.10/glpi and you can start configuring the glpi and follow the instructions , use the user that you created and the database that you created to initiate the glpi on this database )

You'll get default credentials that you can use to connect to the glpi afterwards : 

![GLPI Defaults](/Assets/glpi-default-acc.png)

And you'll get access to the glpi , don't forget to change the defaults passwords : 

![GLPI Menu](/Assets/glpi-menu.png)

Let's configure the back-up disk : 

```
# On the Host: Create a backup disk

qemu-img create -f qcow2 backup_disk.qcow2 10G

# Attach it to the VM 

sudo virsh attach-disk Ubuntu-Server ~/backup_disk.gcow2 vdb

# On the VM : Partition the disk 

sudo fdisk /dev/vdb

# Format the partition

sudo mkfs.ext4 /dev/vdb1

```

Work In Progress - unfinished ! 