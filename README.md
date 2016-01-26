# Configuration of OpenVPN and Easy-RSA v2.2 on a Raspberry PI using ethernet

# Prerequisites
- Assumes you are installing easy-rsa v2.2, v2.3 requires different steps
- This has NOT been tested using wireless (wlan0)
- Open 1194 and enable UDP port forwarding on your router
- Setup static IP address on PI i.e. 10.0.1.20 will be used in this tutorial

# Steps
- Default setup steps
```
sudo apt-get install iptables openvpn
sudo mkdir /etc/openvpn/easy-rsa/
sudo cp -rf /usr/share/doc/openvpn/examples/easy-rsa/2.0/* /etc/openvpn/easy-rsa
sudo chown -R $USER /etc/openvpn/easy-rsa/
```

- Open vars and edit the following values:
```
sudo nano /etc/openvpn/easy-rsa/vars
```

- Set the following values
```
export EASY_RSA="/etc/openvpn/easy-rsa"
export KEY_SIZE=2048
export KEY_COUNTRY="IE"
export KEY_PROVINCE="DU"
export KEY_CITY="Dublin"
export KEY_ORG="Longie"
export KEY_EMAIL="jlongieirl@gmail.com"
```

- More commands
```
cd /etc/openvpn/easy-rsa/
source vars
./clean-all
ln -s openssl-1.0.0.cnf openssl.cnf
```

- Generate your master certificate and key
```
./build-ca
```

- Use the following params:
```
Common name: HomePI
Name: Jlong
Email: jlongieirl@gmail.com
```
Note: 'ca.key' and 'ca.crt' created inside /etc/openvpn/easy-rsa/keys/

- Generate server certificates
```
/etc/openvpn/easy-rsa/build-key-server HomeServerVPN
```
Note: All keys and certificates will be generated in /etc/openvpn/keys folder
<b>KEEP THESE SECURE!!!!</b>

- Add another layer of protection, helps to prevent denial of service (DOS) attacks
```
cd /etc/openvpn/easy-rsa/
openvpn --genkey --secret keys/ta.key
```

- Generate client keys
```
./build-key HomeClientVPN
```
Ensure a password is configured and accept 'Y' to all defaults

- Encrypt the client private key using triple DES
```
cd /etc/openvpn/easy-rsa/keys
openssl rsa -in HomeClientVPN.key -des3 -out HomeClientVPN.3des.key
```

- Generate Diffie Hellman parameters, this one is used by OpenVPN for keys exchange
```
./build-dh
```
Note: if you are lucky this step is fast otherwise it can take a while! Be patient.

Note: Root Certificate, Server Certificate and Client Certificates are now generated. Client files can be used BUT this tutorial will generate an OPVN config file to be used by Android/Tunnelblick applications.

- Export these files if you dont want to have single config file.
```
scp ca.crt HomeClientVPN.key HomeClientVPN.crt root@whereever:/etc/openvpn
```

- Setup Server config
```
sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz /etc/openvpn/
sudo gzip -d /etc/openvpn/server.conf.gz
sudo nano /etc/openvpn/server.conf
```

- Enable packet forwarding for IPv4
```
sysctl -w net.ipv4.ip_forward=1
sysctl -p
```

<!--- Setup iptables-->
<!--```-->
<!--iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE-->
<!--iptables -A FORWARD -p tcp -s 10.8.0.0/24 -d 0.0.0.0/0 -j ACCEPT-->
<!--```-->

- Setup firewall rules to be executed when network interface is loaded
```
sudo nano /etc/firewall-openvpn-rules.sh
```

- Add the following
```
#!/bin/sh 
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j SNAT --to-source 10.0.1.20
```

- Update network interfaces
```
sudo nano vim /etc/network/interfaces
```

- Add the following line to eth0
```
iface eth0 inet static
	pre-up /etc/firewall-openvpn-rules.sh
```

- Startup server
```
sudo /etc/init.d/openvpn start
sudo nano /var/log/openvpn.log
```

- Ensure tunnel is setup correctly
```
ifconfig tun0
```
Should look like the following:
````
tun0      Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  
          inet addr:10.8.0.1  P-t-P:10.8.0.2  Mask:255.255.255.255
          UP POINTOPOINT RUNNING NOARP MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:100 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

# Client files
- Step 1, create default file:
```
nano /etc/openvpn/easy-rsa/keys/Default.txt
```

Add the following:
```
client
dev tun
proto udp
remote YOUR_EXTERNAL_IP_ADDRESS 1194
resolv-retry infinite
nobind
persist-key
persist-tun
mute-replay-warnings
ns-cert-type server
key-direction 1 
cipher AES-128-CBC
comp-lzo
verb 1
mute 20
```

- Download generator file
```
cd /etc/openvpn/easy-rsa/keys
wget --no-check-cert https://gist.githubusercontent.com/laurenorsini/10013430/raw/bd4b64e6ff717dc0d9284081fe3ca096947d0009/MakeOpenVPN.sh -O MakeOVPN.sh
sudo chmod 700 /etc/openvpn/easy-rsa/keys/MakeOVPN.sh
sudo nano /etc/openvpn/easy-rsa/keys/MakeOVPN.sh
sudo ./MakeOVPN.sh
```
When prompted, enter the following:
```
Client: HomeClientVPN
```
Note: HomeClientVPN.ovpn is now generated and can exported allowing you to connect using Android or Tunnelblick

- Export file
```
scp HomeClientVPN.ovpn root@host:/Users/jlong/Documents/VPN
```

- Ensure everything works at startup!!!
```
sudo reboot
sudo nano /var/log/openvpn.log
```

Having exported HomeClientVPN.ovpn, use it to confirm your client config is working. Once connected, a good way to ensure your connection is encrypted, open http://www.whatsmyip.org/ and ensure the IP address is that of your home IP address.

# Adding more clients
```
cd /etc/openvpn/easy-rsa/keys
./build-key Mates-ClientVPN
openssl rsa -in Mates-ClientVPN.key -des3 -out Mates-ClientVPN.3des.key
sudo ./MakeOVPN.sh
```
when prompted type in 'Mates-ClientVPN' and then send to your mate, dont forget the password used when you created the config!!!
```
scp Mates-ClientVPN.ovpn root@host:/Users/jlong/Documents/VPN
```


