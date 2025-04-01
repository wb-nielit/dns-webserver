# **DNS & Web Server Setup on Ubuntu**
## **Documented By : [Mr. Rahul](https://github.com/cyb-sehgal) (Cyber Security Trainer)**  
### **Objective**  
This project focuses on setting up a Domain Name System (DNS) server using BIND9 on an Ubuntu machine. The goal is to configure a static IP, assign a Fully Qualified Domain Name (FQDN), install and configure BIND9, create forward lookup zones, and validate the DNS setup.  

### **Key Features**  
- Assigning a **static IP** to ensure network stability.  
- Setting up a **Fully Qualified Domain Name (FQDN)** for hostname resolution.  
- Installing **BIND9** as the DNS server and configuring it to work in **IPv4 mode**.  
- Configuring **BIND9 settings**, including `named.conf.options` and `named.conf.local`.  
- Creating a **forward lookup zone file** for resolving domain names to IP addresses.  
- **Testing the DNS setup** using `nslookup` & `windows client` and verifying that queries return the correct IPs.  

### **Use Case**  
This setup is useful in enterprise environments where internal DNS servers are needed to manage local domain resolution efficiently. It can also be used for **web hosting**, **private cloud services**, or **penetration testing labs** requiring custom domain configurations.  
 
---

## **Assigning Static IP to the Ubuntu Machine**  

### **Checking the Current IP Allocation**  
Run the following command:  
```bash
ip r
```
This will show if the machine is using DHCP.

### **Making the IP Static**  
1. Navigate to the netplan directory:  
   ```bash
   cd /etc/netplan
   ls
   ```
   You'll see a file named `01-network-manager-all.yaml`.

2. **Backup the file before making changes:**  
   ```bash
   sudo mv 01-network-manager-all.yaml 01-network-manager-all.yaml-bak
   ```
   
3. **Create a new configuration file:**  
   ```bash
   sudo nano 00-installer-config.yaml
   ```
   
4. **Add the following configuration:**  
   ```yaml
   network:
     renderer: networkd
     ethernets:
       ens33:
         addresses:
           - 192.168.116.132/24  #Your machine IP (check using 'ip a')
         nameservers:
           addresses: [4.2.2.2, 8.8.8.8]
         routes:
           - to: default
             via: 192.168.116.2  #Your machine gateway (check using 'ip r')
     version: 2
   ```
   
5. **Verify changes:**  
   ```bash
   cat 00-installer-config.yaml
   ```

6. **To apply changes.**  
   ```bash
   sudo netplan apply
   ```

---

## **Setting a Fully Qualified Domain Name (FQDN)**  
1. Check the current hostname:  
   ```bash
   hostname
   ```

2. Modify the `/etc/hosts` file:  
   ```bash
   sudo nano /etc/hosts
   ```
   Add these line with your system's IP and desired FQDN.
   Like this:
   ```bash
    127.0.0.1 localhost
    127.0.0.1 dns.nielitinfosec.local Ubuntu
    192.168.116.132 dns.nielitinfosec.local Ubuntu

    # The following lines are desirable for IPv6 capable hosts
    ::1     ip6-localhost ip6-loopback
    fe00::0 ip6-localnet
    ff00::0 ip6-mcastprefix
    ff02::1 ip6-allnodes
    ff02::2 ip6-allrouters
    ```
    _⚠️ In my case **Ubuntu** is the Hostname you have to add your own Hostname accordingly._    
    Now restart the systemd-hostnamed to apply the chanegs immediately
    ```bash
    sudo systemctl restart systemd-hostnamed
    ```
4. **Verify the changes:**  
   ```bash
   hostname
   hostname --fqdn
   ```

---

## **BIND9 Installation on Ubuntu**  

### **Updating the Repositories**  
```bash
sudo apt update
```

### **Installing BIND9 and Utilities**  
```bash
sudo apt install bind9 bind9utils bind9-doc -y
```

### **Forcing IPv4 Mode for BIND9**  
Modify `/etc/default/named` to include `-4` for IPv4 mode:  
```bash
sudo nano /etc/default/named
```
Add `-4` in the appropriate place, then save and exit.
something like this
```bash
#
# run resolvconf?
RESOLVCONF=no

# startup options for the server
OPTIONS="-4 -u bind"
```
### **Restarting BIND9**  
```bash
sudo systemctl restart bind9
```

### **Checking if Port 53 is Listening**  
```bash
netstat -antu | grep 53
```
You will see something like this
```bash
tcp        0      0 127.0.0.54:53           0.0.0.0:*               LISTEN     
tcp        0      0 192.168.116.136:53      0.0.0.0:*               LISTEN     
tcp        0      0 192.168.116.136:53      0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN     

```
If port 53 is not listening, allow it through the firewall:  
```bash
sudo ufw allow bind9
```

---

## **BIND9 Configuration**  

1. **Navigate to the configuration directory:**  
   ```bash
   cd /etc/bind
   ```

2. **View the main configuration file:**  
   ```bash
   sudo nano named.conf
   ```
   _⚠️ Remember You Don't have modify this file. Instead edit **named.conf.options**_

3. **Edit `named.conf.options`:**  
   ```bash
   sudo nano named.conf.options
   ```
   Erase the default content and Add these following lines:  
   ```bash
   acl "trusted-clients" {
      192.168.40.0/24;  // Entire subnet
   };

   options {
      directory "/var/cache/bind";

      // Allow queries from the entire 192.168.40.x network
      allow-query {
         localhost;
         trusted-clients;
      };

      forwarders {
         8.8.8.8;
         8.8.4.4;
      };

      recursion yes;
   };

   ```
   _⚠️ Remember You have to add your own Subnet here!_    

   Save and exit.

  _💡 To check for sysatax errors in the configuration file use this command_   
  ```bash
  sudo named-checkconf named.conf.options
  ```   
   _⚠️ After execution. if the syntax is correct, the returned result will be wihtout any message!_   

4. **Edit `named.conf.local` to define the zone file:**  
   ```bash
   sudo nano named.conf.local
   ```
   Erase the default content and Add these following lines:  
   ```bash
   // Define the Forward zone
   // My domain: www.nielitinfosec.local
   // Forward file called fwd.nielitinfosec.local
   zone "nielitinfosec.local" IN { 
      type master;
      // Path of Forward file
      file "/etc/bind/nielit/fwd.nielitinfosec.local";
   };
   // Define the Reverse zone
   // Reverse file called: rvs.nielitinfosec.local
   zone "1.168.192.in-addr.arpa" IN {
         type master;
         // Path of Reverse file
         file "/etc/bind/nielit/rvs.nielitinfosec.local";
   };

   ```
In my case The directory ```nielit``` and the configuration file ```fwd.nielitinfosec.local``` will be created later! you have to create one accordingly.    

And same for the ```rvs.neilitinfosec.local``` in the directory ```nielit```     

Now save the file and exit.     

💡 To check for the sysatax errors in the configuration file, use this command:
```bash
sudo named-checkconf named.conf.local
```
  _⚠️ After execution. if the syntax is correct, the returned result will be wihtout any message!_  

5. **Create the zones directory and zone file:**  
```bash
   sudo mkdir nielit
   cd nielit
```  
   In this **_directory,_** create a configuration file for the Forward Zone.    
```bash
sudo nano fwd.nielitinfosec.local
```

6. **Add the following lines in the _forward.neilitinfosec.local_ configuration file:**  
```bash
$TTL    604800
; SOA record with MNAME and RNAME updated
@       IN      SOA     nielitinfosec.local. root.nielitinfosec.local. (
                              3         ; Serial Note: increment after each change
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
; Name server record 
@       IN      NS      dns.nielitinfosec.local.
; A record for name server
dns     IN      A       192.168.40.128

; Mail handler or MX record for the domain nielitinfosec.local
nielitinfosec.local.    IN     MX   10   mail.nielitinfosec.local.

; A rocerd for Mail server
mail    IN      A       192.168.40.130
; A record for web server
www     IN      A       192.168.40.128

```   
Save and exit.    

💡 To check for the sysatax errors in the configuration file, use this command:  
```bash
sudo named-checkzone nielitinfosec.local fwd.nielitinfosec.local
```
_⚠️ After execution.! You'll see something like this:_  
```bash
zone nielitinfosec.local/IN: loaded serial 3
OK
``` 
7. In the same **_directory,_** create a configuration file for the Reverse Zone.
```bash
sudo nano rvs.neilitinfosec.local
```

8. **Add the following lines in the _reverse.neilitinfosec.local_ configuration file:**
```bash
$TTL    604800
; SOA record with MNAME and RNAME updated
@       IN      SOA     nielitinfosec.local. root.nielitinfosec.local. (
                              2         ; Serial Note: increment after each change
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
; Name server record 
@        IN      NS      dns.nielitinfosec.local.
; A record for Name Server
dns      IN      A       192.168.40.128
; PTR record for Name Server
128      IN      PTR     dns.nielitinfosec.local.
; PTR record for Mail Server
130      IN      PTR     mail.nielitinfosec.local.
; PTR record for Web Server
128      IN      PTR     www.nielitinfosec.local.


``` 
Save and exit!  

💡 To check for the sysatax errors in the configuration file, use this command:
```bash
sudo named-checkzone nielitinfosec.local rvs.nielitinfosec.local
```
_⚠️ After execution.! You'll see something like this:_  
```bash
zone nielitinfosec.local/IN: loaded serial 2
OK
```

## **Verifying & Restarting BIND9**  

1. **Check configuration syntax:**  
```bash
sudo named-checkconf
```
_⚠️ After execution. if the configuration is correct, the returned result will be wihtout any message!_ 

2. **Restart BIND9 to apply changes:**  
   ```bash
   sudo systemctl restart bind9
   ```

**_⚠ Skip this step if you're not using a firewall.!_**

3. **Configurng the Bind9 service to be allowed through the firewall!**
```bash
sudo ufw status
```     
```bash
sudo ufw allow Bind9
```     
After adding the rule, Reload the UFW
```bash
sudo ufw reload
```     
After reloading the firewall, check the firewall status:    
```bash
sudo ufw status
```
## Configuring the WEB server
1. **Updating the Repositories:**
```bash
sudo apt update 
```
2. **Install Apache2**
```bash
sudo apt install apache2 -y
```
3. **After installation, start and enable Apache to run at boot:**
```bash
sudo systemctl start apache2
sudo systemctl enable apache2
```
4. **Check its status to ensure it's running:**
```bash
sudo systemctl status apache2
```
5. **Allow External Access to Apache (Firewall)**
_⚠ By default, Apache2 listens on port 80 (HTTP). You need to allow traffic from your Windows client:_
```bash
sudo ufw allow 80/tcp
sudo ufw reload
```
6. **Configure Apache**
_⚠ The default web root directory is:_
```bash
cd /var/www/html
```
7. **You can create or modify your own website files here:**
_⚠ After the changes you have to restart the apache services!_
```bash
sudo systemctl restart apache2
```
## Configuring the DNS on the Windows client:
   * Press Win logo button + R
   * Type ```ncpa.cpl``` and press Enter
   * In the Network Connection choose your Network Adapter & Double click on it (e.g.: Ethernet or Wi-Fi)
   * Go to properties
   * Scroll down and choose: ```Internet Protocol Version 4 (TCP/IP)``` Double click on it
   * Here, choose the Options: ```Use the following DNS server address:```
   * In the ```Preferred DNS server:``` Enter the IP address of DNS Server (Ubuntu Machine) and click on OK
   * Double click on the same ```Adapter``` you just configured, Disable it and Enable it to apply the chanegs
   * Close the Control Panel
## **Testing on windows client**
1. use the **_ping_** command in CMD.
```bash
ping 192.168.116.132
```     
_⚠️ After execution.! You'll see something like this:_  
```bash
Pinging 192.168.116.132 with 32 bytes of data:
Reply from 192.168.116.132: bytes=32 time=1ms TTL=64
Reply from 192.168.116.132: bytes=32 time<1ms TTL=64
Reply from 192.168.116.132: bytes=32 time=1ms TTL=64
Reply from 192.168.116.132: bytes=32 time=1ms TTL=64
```     
✔ this means you're successfully connected to the DNS Sever.!
## **Testing DNS Server**
```bash
ping www.nielitinfosec.local
```
_⚠️ After execution.! You'll see something like this:_
```bash
Pinging www.nielitinfosec.local [192.168.40.128] with 32 bytes of data:
Reply from 192.168.40.128: bytes=32 time=1ms TTL=64
Reply from 192.168.40.128: bytes=32 time<1ms TTL=64
Reply from 192.168.40.128: bytes=32 time=2ms TTL=64
Reply from 192.168.40.128: bytes=32 time<1ms TTL=64
```
## 💡 If you browse this domain from windows client
```bash
www.nielitinfosec.local
```
## You'll See something like this:

![nielitinfosec](https://github.com/user-attachments/assets/3c3c5bdb-635c-49f6-b3dd-5b9b046173fc)
