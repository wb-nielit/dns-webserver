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
   127.0.1.1 server.infosec.local Ubuntu
   192.168.116.136 server.infosec.local Ubuntu

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
sudo apt install bind9 bind9utils bind9-doc
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
   Add the following lines:  
   ```bash
   recursion yes;
   listen-on port 53 { 192.168.116.132; };
   allow-query { any; };
   ```
   _⚠️ Remember You have to add your own IP here_   
   Save and exit.

4. **Edit `named.conf.local` to define the zone file:**  
   ```bash
   sudo nano named.conf.local
   ```
   Add:  
   ```bash
   zone "nielitinfosec.com" {
       type master;
       file "/etc/bind/zones/forward.zone.nielitinfosec.com";
   };
   ```

5. **Create the zones directory and zone file:**  
   ```bash
   sudo mkdir zones
   sudo nano /etc/bind/zones/forward.zone.nielitinfosec.com
   ```

6. **Add the following zone file configuration:**  
   ```bash
   ; BIND forward zone file for nielitinfosec.com
   $TTL 2d
   $ORIGIN nielitinfosec.com.
   @       IN      SOA     dns.nielitinfosec.com. admin.nielitinfosec.com. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
           IN      NS      dns.nielitinfosec.com.
   dns     IN      A       192.168.116.132
   mail    IN      A       192.168.116.135
   pay     IN      A       192.168.116.136
   cloud   IN      A       192.168.116.137
   ```
   Save and exit.

---

## **Verifying & Restarting BIND9**  

1. **Check configuration syntax:**  
   ```bash
   sudo named-checkconf
   ```
   If no errors are displayed, your configuration is correct.

2. **Check the zone file:**  
   ```bash
   sudo named-checkzone nielitinfosec.com /etc/bind/zones/forward.zone.nielitinfosec.com
   ```
   If it returns "OK", the configuration is correct.

3. **Restart BIND9 to apply changes:**  
   ```bash
   sudo systemctl restart bind9
   ```

---

## **Testing DNS Server**  

1. **Use `nslookup` to test the DNS resolution:**  
   ```bash
   nslookup mail.nielitinfosec.com
   nslookup cloud.nielitinfosec.com
   ```
   If the configuration is correct, it will return the respective IP addresses.

---

This is the extracted and well-structured version of your PDF. Let me know if you need any modifications or additional formatting! 🚀
