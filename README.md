In order to install the OpenVPN package, we'll first need to install the EPEL repo:
```[root@Server1]# yum -y install epel-release```
Once EPEL is installed, we can go ahead with installing OpenVPN:
```[root@Server1]# yum -y install openvpn```
Let's enable masquerading in the firewall, and then reload things so the changes take effect:
```
[root@Server1]# firewall-cmd --permanent --add-port=1194/tcp
[root@Server1]# firewall-cmd --permanent --add-masquerade
[root@Server1]# firewall-cmd --reload
```

Create Keys and Credentials on `Server1`
We'll use EasyRSA to create and sign the keys for the server and client. Install it with this:
```[root@Server1]# yum -y install easy-rsa```
Create a directory to hold the files we'll create:
```[root@Server1]# mkdir /etc/openvpn/easy-rsa```
and change our working directory to it:
```[root@Server1]# cd /etc/openvpn/easy-rsa```
To make things a littler easier, let's append the EasyRSA executable folder to our current path:
```[root@Server1]# PATH=$PATH:/usr/share/easy-rsa/3.0.7/```
Initialize PKI:
```[root@Server1]# easyrsa init-pki```
Build the CA (remember the password you use, you can leave the common name as the default):
```[root@Server1]# easyrsa build-ca```
Generate a Diffie-Hellman key for forward secrecy:
```[root@Server1]# easyrsa gen-dh```
Now we'll move on to the server credentials. For convenience, we won’t password protect these.
Create the server certificate:
```[root@Server1]# easyrsa gen-req server nopass```
Sign the server certificate:
```[root@Server1]# easyrsa sign-req server server```
We'll be prompted to type yes here. There's also a spot in here where we've got to enter the password we created a few steps back, with the easyrsa init-pki command.
Create the client certificate:
```[root@Server1]# easyrsa gen-req client nopass```
Sign the client certificate:
```[root@Server1]# easyrsa sign-req client client```
Type yes when prompted, and enter the same pass we did for the server creation.
Now we need to generate the TLS key:
```
[root@Server1]# cd /etc/openvpn
[root@Server1]# openvpn --genkey --secret pfs.key
```

Configure the OpenVPN Server on `Server1`
You'll need to create and edit /etc/openvpn/server.conf:
```
[root@Server1]# vim /etc/openvpn/server.conf
port 1194
proto tcp
dev tun
ca /etc/openvpn/easy-rsa/pki/ca.crt
cert /etc/openvpn/easy-rsa/pki/issued/server.crt
key /etc/openvpn/easy-rsa/pki/private/server.key
dh /etc/openvpn/easy-rsa/pki/dh.pem
topology subnet
cipher AES-256-CBC
auth SHA512
server 10.8.0.0 255.255.255.0
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"
ifconfig-pool-persist ipp.txt
keepalive 10 120
comp-lzo

persist-key
persist-tun
status openvpn-status.log
log-append openvpn.log
verb 3
tls-server
tls-auth /etc/openvpn/pfs.key```
Now you can enable and start OpenVPN:
```[root@Server1]# systemctl enable openvpn@server.service```
```[root@Server1]# systemctl start openvpn@server.service```
Package up Keys and Certificates on `Server1` for Copying to `Client1`
You'll need to package up the credentials we created, and copy them to Client1, you can do 
this by creating the following shell script:
```[root@Server1]# vim keys.sh
cd /etc/openvpn
mkdir -p server1/keys
cp pfs.key server1/keys
cp easy-rsa/pki/dh.pem server1/keys
cp easy-rsa/pki/ca.crt server1/keys
cp easy-rsa/pki/private/ca.key server1/keys
cp easy-rsa/pki/private/client.key server1/keys
cp easy-rsa/pki/issued/client.crt server1/keys
tar cvzf /tmp/keys.tgz server1/```
Make it executable:
```[root@Server1]# chmod +x keys.sh```
And run it:
```[root@Server1]# ./keys.sh```
Install OpenVPN on `Client1`
Just like on Server1, you'll need to install EPEL before you can install OpenVPN:
```[root@Client1]# yum -y install epel-release
[root@Client1]# yum -y install openvpn```
Copy and Install Keys from `Server1` to `Client1`
Now we need to copy the keys we tarred up on Server1 over to Client1.
On Client1:
```[root@Client1]# cd /etc/openvpn`
[root@Client1]# scp cloud_user@10.0.1.10:/tmp/keys.tgz ./```
We'll need the password for Server1 at that point. Once the tar file makes the trip, we can extract it:
```[root@Client1]# tar xvzf keys.tgz```
check_circleConfigure the VPN client on `Client1`
With the keys in place, we can configure the client:
```[root@Client1]# vim client.conf
client
dev tun
proto tcp
remote <tun ip address>1194  
ca server1/keys/ca.crt
cert server1/keys/client.crt
key server1/keys/client.key
tls-version-min 1.2
tls-cipher TLS-ECDHE-RSA-WITH-AES-128-GCM-SHA256:TLS-ECDHE-ECDSA-WITH-AES-128-GCM-SHA256:TLS-ECDHE-RSA-WITH-AES-256-GCM-SHA384:TLS-DHE-RSA-WITH-AES-256-CBC-SHA256
cipher AES-256-CBC
auth SHA512
resolv-retry infinite
auth-retry none
nobind
route-nopull
persist-key
persist-tun
ns-cert-type server
comp-lzo
verb 3
tls-client
tls-auth server1/keys/pfs.key```
Start the Client:
```[root@Client1]# systemctl start openvpn@client.service```

Add a Static Route on Client1
In order to have Client1 traffic to node1 originate on the 10.8.0.0/24 network, we'll need to add a static route, so that the VPN tunnel is the interface that connects to that host:
```[root@Client1]# ip route add 10.0.1.20 dev tun0```
We can can verify the entry using:
```[root@Client1]# ip route show```
We should now be able to access the website on node1:
```[root@Client1]# curl 10.0.1.20```
