# Configuration of OpenVPN and Easy-RSA v2.0 on a Raspberry PI

Wizard setup:
http://kamilslab.com/2017/01/22/how-to-turn-your-raspberry-pi-into-a-home-vpn-server-using-pivpn/


# Prerequisites
- Assumes you are installing easy-rsa v2 which no longer has the easy-rsa bundled as part of the install
- Do NOT setup this against the wireless card (wlan0)
- Tested on a RaspPI Model 2
- Open 1194 and enable UDP port forwarding on your router to the IP address 10.0.1.20
- You have obtained the following IP addresses, required during the install
```
Gateway/Internet Router: 10.0.1.1
Subnet Mask: 255.255.255.0
Static IP: 10.0.1.20
Router gives out DHCP range: 10.0.100-200
dns-nameservers 10.0.1.1
```

# Initial Steps
- Download, configure and setup environment, easy-rsa is no longer available in the openvpn bundle
```
sudo apt-get -y install iptables openvpn git wget curl
sudo mkdir -p /etc/openvpn/easy-rsa/
git clone -b release/2.x https://github.com/OpenVPN/easy-rsa.git ~/easy-rsa
sudo cp -rf ~/easy-rsa/easy-rsa/2.0/* /etc/openvpn/easy-rsa
sudo chown -R $USER /etc/openvpn/easy-rsa/
```

- Open vars
```
sudo vi /etc/openvpn/easy-rsa/vars
```

- Update the following values
```
export EASY_RSA="/etc/openvpn/easy-rsa"
export KEY_SIZE=2048
export KEY_COUNTRY="IE"
export KEY_PROVINCE="DU"
export KEY_CITY="Dublin"
export KEY_ORG="Longie"
export KEY_EMAIL="jlongieirl@gmail.com"
export KEY_OU="ITDept"
```

- This is a fresh install so we'll remove any previous keys
```
cd /etc/openvpn/easy-rsa/
source vars
./clean-all
ln -s openssl-1.0.0.cnf openssl.cnf
```

- Generate master certificate and key, selecting enter for all options. These will become your certificate authority keys.
```
./build-ca # change the values where required
```
__Note:__ 'ca.key' and 'ca.crt' is created inside in the folder /etc/openvpn/easy-rsa/keys/


- Generate server certificates, selecting enter for all and __DO NOT__ enter a challenge password. Say y to sigining of the certificate and to the commit prompt. This will create HomeServerVPN.key, HomeServerVPN.csr and HomeServerVPN.crt in /etc/openvpn/easy-rsa/keys folder
```
/etc/openvpn/easy-rsa/build-key-server HomeServerVPN
```

<b>KEEP THESE KEYS SECURE!!!!</b>

- Add another layer of protection, helps to prevent denial of service (DoS) attacks. The server will not try to authenticate an access request if it does not detect static HMAC key.
```
cd /etc/openvpn/easy-rsa/
openvpn --genkey --secret keys/ta.key
```

- Generate client keys, selecting enter for all options except those where changeme is specified. __DO NOT__ set a challenge password and say Y to the commit prompt.
```
/etc/openvpn/easy-rsa/build-key HomeClientVPN
```

- Encrypt the client private key using triple DES, please __REMEMBER__ the password as you will need it later when connecting through a device. If no password is created then the steps below __WILL NOT__ work!
```
cd keys
openssl rsa -in HomeClientVPN.key -des3 -out HomeClientVPN.3des.key
```

- Generate Diffie Hellman parameters, this one is used by OpenVPN for keys exchange
```
cd ..
./build-dh
```

__Note:__ Be patient....it will finish, since we are using 2048-bit encryption it can take an hour, if you use 1024 it will take five minutes but is less secure.

- Root Certificate, Server Certificate and Client Certificates are now generated. Client files can be used __BUT__ this tutorial will generate an OPVN config file to be used by Android/iOS/Tunnelblick applications which will combine all three into one. 

- You can export them if you dont want to generate an OVPN file
```
cd /etc/openvpn/easy-rsa/keys
scp ca.crt HomeClientVPN.key HomeClientVPN.crt root@some_other_server:/Users/jlong/Documents/secure
```

- Generate server.conf, replacing 10.0.1.20 with the IP address of your PI machine
```
sudo cat << EOF > /etc/openvpn/server.conf
# IP address of PI 
local 10.0.1.20 255.255.255.0
dev tun
proto udp
port 1194
# Certs and Keys generated
ca /etc/openvpn/easy-rsa/keys/ca.crt
cert /etc/openvpn/easy-rsa/keys/HomeServerVPN.crt
key /etc/openvpn/easy-rsa/keys/HomeServerVPN.key
dh /etc/openvpn/easy-rsa/keys/dh2048.pem
# Your default VPN network, just make sure it doesnt clash with any other internal network IP range
server 10.8.0.0 255.255.255.0
# Your default server and remote endpoints, again ensure they dont clash with any other internal network IP range
ifconfig 10.8.0.1 10.8.0.2
# Add route to Client routing table for the OpenVPN Server
push "route 10.8.0.1 255.255.255.255" 
# Add route to Client routing table for the OpenVPN Subnet
push "route 10.8.0.0 255.255.255.0" 
# Your local subnet 
push "route 10.0.1.20 255.255.255.0" # YOUR PI’S IP ADDRESS
# Set primary domain name server address to the SOHO Router
# If your router does not do DNS, you can use Google DNS 8.8.8.8
# Configure your routers IP address on the network
push "dhcp-option DNS 10.0.1.1" 
# Google DNS addresses.Use this if you are having issues with local DNS routing
#push "dhcp-option DNS 8.8.8.8"
#push "dhcp-option DNS 8.8.4.4"
# Override the Client default gateway by using 0.0.0.0/1 and
# 128.0.0.0/1 rather than 0.0.0.0/0. This has the benefit of 
# overriding but not wiping out the original default gateway.
push "redirect-gateway def1" 
client-to-client
duplicate-cn
keepalive 10 120
tls-auth /etc/openvpn/easy-rsa/keys/ta.key 0
cipher AES-128-CBC
comp-lzo
user nobody
group nogroup
persist-key
persist-tun
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3
EOF
```

- Enable packet forwarding for IPv4, this will allow your PI machine to act as relay to the internet. If you want to only access to your local network, then leave this step out.
```
sudo /bin/su -c "echo -e '\n#Enable IP Routing\nnet.ipv4.ip_forward = 1' >> /etc/sysctl.conf"
sudo sysctl -p
```

- IPv4 Packet forwarding is now enabled!
```
kernel.printk = 3 4 1 3
vm.swappiness = 1
vm.min_free_kbytes = 8192
net.ipv4.ip_forward = 1
```

- Setup firewall rules to be executed when network interface is loaded
```
sudo vi /etc/firewall-openvpn-rules.sh
```

- Add the following, changing 'to-source' to the IP address of your PI machine
```
#!/bin/sh 
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j SNAT --to-source 10.0.1.20
```

- Lock down the file
```
sudo chmod 700 /etc/firewall-openvpn-rules.sh 
sudo chown root /etc/firewall-openvpn-rules.sh
```

- Update network interfaces, open interfaces and ensure the eth0 is static and not manual. I would suggest locking the IP address used in your router to the MAC address of your PI!
```
sudo vi /etc/network/interfaces
```

- Add new OpenVPN rule
```
iface eth0 inet static
    pre-up /etc/firewall-openvpn-rules.sh
```

- Ensure the rules are applied at startup
```
sudo sed -i -e '$i \/etc/firewall-openvpn-rules.sh\n' /etc/rc.local
```

- Startup server and sure the logs are running as expected
```
sudo /etc/init.d/openvpn restart
```

- Ensure tunnel is setup correctly
```
ifconfig tun0
```

- You should see the following output
````
tun0      Link encap:UNSPEC  HWaddr 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  
          inet addr:10.8.0.1  P-t-P:10.8.0.2  Mask:255.255.255.255
          UP POINTOPOINT RUNNING NOARP MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:100 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

- Any issues, tail the log for root causes. In the majority cases its a mis-spelt word or incorrect directory location
```
sudo tail -f /var/log/openvpn.log
sudo tail -f /var/log/openvpn-status.log
```

# Client files
- Copy the entire text, this will create the detault params required. It assumes your machine has internet connectivity to run this!
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
key-direction 1
cipher AES-128-CBC
comp-lzo
verb 1
mute 20
EOF
```

- Download MakeOVPN generator, when prompted, enter the following: 'HomeClientVPN'
```
cd /etc/openvpn/easy-rsa/keys
wget --no-check-cert https://gist.githubusercontent.com/laurenorsini/10013430/raw/bd4b64e6ff717dc0d9284081fe3ca096947d0009/MakeOpenVPN.sh -O MakeOVPN.sh
sudo chmod 700 /etc/openvpn/easy-rsa/keys/MakeOVPN.sh
./MakeOVPN.sh
```
__Note:__ HomeClientVPN.ovpn is now generated and can exported allowing you to connect using Android/iOS or Tunnelblick

- Export client file to a location to be consumed by devices
```
scp HomeClientVPN.ovpn root@some_other_server:/Users/user/folder
```

# Restart Machine
- Reboot your PI to ensure all the firewall rules are applied and ther OpenVPN service boots up as expected:
```
sudo reboot
```

- On restart, ensure everything is working as expected by connecting to the VPN via your chosen device. If you have connected successfully, you will see the connection logged
```
sudo tail -f /var/log/openvpn.log
```

# Adding more clients
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
