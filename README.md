# Configuration of OpenVPN and Easy-RSA v2.0 on a Raspberry PI using ethernet

# Prerequisites
- Assumes you are installing easy-rsa v2, different steps for v3
- This has NOT been tested using wireless (wlan0)
- Setup static IP address i.e. 10.0.1.20 will be used in this tutorial. The router gives the IP address based on the MAC address of the device 'ifconfig eth0'
- Open 1194 and enable UDP port forwarding on your router to the IP address 10.0.1.20

# Initial Steps
- Step1. Download and install required packages
```
sudo apt-get install iptables openvpn git wget curl
sudo mkdir /etc/openvpn/easy-rsa/
sudo cp -rf /usr/share/doc/openvpn/examples/easy-rsa/2.0/* /etc/openvpn/easy-rsa
```

- Step2. If the examples/easy-rsa folder is not available, then download and extract it from the OpenVPN repo
```
git clone -b release/2.x https://github.com/OpenVPN/easy-rsa.git ~/easy-rsa
sudo cp -rf ~/easy-rsa/easy-rsa/2.0/* /etc/openvpn/easy-rsa
```

- Step3. Easy-rsa folder is setup, make it available to the logged in user! Ensure you are NOT logged in as root when doing this task!
```
sudo chown -R $USER /etc/openvpn/easy-rsa/
```

- Open vars and edit the following values:
```
sudo vi /etc/openvpn/easy-rsa/vars

export EASY_RSA="/etc/openvpn/easy-rsa"
export KEY_SIZE=2048
export KEY_COUNTRY="IE"
export KEY_PROVINCE="DU"
export KEY_CITY="Dublin"
export KEY_ORG="Longie"
export KEY_EMAIL="jlongieirl@gmail.com"
export KEY_OU="ITDept"
```

- More commands, export environment variables and delete any previously created certificates. This is a fresh install so we'll remove any previous keys.
```
cd /etc/openvpn/easy-rsa/
source vars
./clean-all
ln -s openssl-1.0.0.cnf openssl.cnf
```

- Generate master certificate and key, selecting enter for all options. These are your certificate authority keys.
```
./build-ca
```

__Note:__ 'ca.key' and 'ca.crt' is created inside /etc/openvpn/easy-rsa/keys/

- Generate server certificates, selecting enter for all and __DO NOT__ enter a challenge password
```
/etc/openvpn/easy-rsa/build-key-server HomeServerVPN
```
Note: All keys and certificates will be generated in /etc/openvpn/keys folder. Please note the name of the server used here.

<b>KEEP THESE KEYS SECURE!!!!</b>

- Add another layer of protection, helps to prevent denial of service (DoS) attacks. The server will not try to authenticate an access request if it does not detect static HMAC key.
```
cd /etc/openvpn/easy-rsa/
openvpn --genkey --secret keys/ta.key
```

- Generate client keys, selecting enter for all options and __DO NOT__ set a challenge password 
```
/etc/openvpn/easy-rsa/build-key HomeClientVPN
```

- Encrypt the client private key using triple DES, please remember the password used here. If no password is created then the steps below wont work!
```
cd /etc/openvpn/easy-rsa/keys
openssl rsa -in HomeClientVPN.key -des3 -out HomeClientVPN.3des.key
```

- Generate Diffie Hellman parameters, this one is used by OpenVPN for keys exchange
```
cd /etc/openvpn/easy-rsa/
./build-dh
```
__Note:__ Be patient....it will finish, since we are using 2048-bit encryption it can take an hour, if you use 1024 it will take five minutes.

- Root Certificate, Server Certificate and Client Certificates are now generated. Client files can be used BUT this tutorial will generate an OPVN config file to be used by Android/Tunnelblick applications which will combine all three into one. So export them for later!
```
cd /etc/openvpn/easy-rsa/keys
scp ca.crt HomeClientVPN.key HomeClientVPN.crt root@some_other_server:/Users/jlong/Documents/secure
```

# Setup Server
- Copy over a default template
```
sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz /etc/openvpn/
sudo gzip -d /etc/openvpn/server.conf.gz
sudo vi /etc/openvpn/server.conf
```

- Replace 10.0.1.20 with the IP address of your machine
__Note:__ 8.8.8.8 is the DNS for Google and this setup is using 2048 bit encryption
```
local 10.0.1.20
dh dh2048.pem
push "route 10.0.1.20 255.255.255.0"
push "dhcp-option DNS 8.8.8.8"
log /var/log/openvpn.log
ca /etc/openvpn/easy-rsa/keys/ca.crt
cert /etc/openvpn/easy-rsa/keys/HomeServerVPN.crt
key /etc/openvpn/easy-rsa/keys/HomeServerVPN.key
dh /etc/openvpn/easy-rsa/keys/dh2048.pem
```

- Enable packet forwarding for IPv4, this will allow your device to act as relay to the internet. If you want to only access your local network, then leave this step out.
```
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -p
```

<!--- Setup iptables-->
<!--```-->
<!--iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE-->
<!--iptables -A FORWARD -p tcp -s 10.8.0.0/24 -d 0.0.0.0/0 -j ACCEPT-->
<!--```-->

- Setup firewall rules to be executed when network interface is loaded
```
sudo vi /etc/firewall-openvpn-rules.sh
```

- Add the following, changing 'to-source' to the IP address of your machine
```
#!/bin/sh 
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j SNAT --to-source 10.0.1.20
```

- Lock down the file
```
sudo chmod +x /etc/firewall-openvpn-rules.sh
sudo chown root /etc/firewall-openvpn-rules.sh
```

- Update network interfaces, find the following line and ensure the eth0 is static and not manual
```
sudo vi /etc/network/interfaces

iface eth0 inet static
	pre-up /etc/firewall-openvpn-rules.sh
```

- Startup server and sure the logs are running as expected
```
sudo /etc/init.d/openvpn start
sudo tail -f /var/log/openvpn.log
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
- Copy the entire text, this will create the detault params. It assumes your machine has internet connectivity to run this.
```
cat << EOF > /etc/openvpn/easy-rsa/keys/Default.txt
client
dev tun
proto udp
remote `curl -s http://myip.enix.org/REMOTE_ADDR` 1194
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
EOF
```

- Download generator file
```
cd /etc/openvpn/easy-rsa/keys
wget --no-check-cert https://gist.githubusercontent.com/laurenorsini/10013430/raw/bd4b64e6ff717dc0d9284081fe3ca096947d0009/MakeOpenVPN.sh -O MakeOVPN.sh
sudo chmod 700 /etc/openvpn/easy-rsa/keys/MakeOVPN.sh
sudo ./MakeOVPN.sh
```

- When prompted, enter the following:
```
Client: HomeClientVPN
```
__Note:__ HomeClientVPN.ovpn is now generated and can exported allowing you to connect using Android or Tunnelblick

- Export file
```
scp HomeClientVPN.ovpn root@some_other_server:/Users/jlong/Documents/VPN
```

# Ensure everything works at startup!!!
```
sudo reboot
sudo cat /var/log/openvpn.log
```

# Add more clients
Having exported HomeClientVPN.ovpn, use it to confirm your client config is working. Once connected, a good way to ensure your connection is encrypted, open http://www.whatsmyip.org/ and ensure the IP address is that of your home IP address.
```
cd /etc/openvpn/easy-rsa/keys
./build-key Mates-ClientVPN
openssl rsa -in Mates-ClientVPN.key -des3 -out Mates-ClientVPN.3des.key
sudo ./MakeOVPN.sh
```

- When prompted type in 'Mates-ClientVPN' and then send to your mate, dont forget the password used when you created the config!!!
```
scp Mates-ClientVPN.ovpn root@host:/Users/jlong/Documents/VPN
```
