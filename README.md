

In order to install the OpenVPN package, you first need to install the EPEL repo:

On OPENVPN-SERVER:

yum -y install epel-release

Once EPEL is installed, you can install OpenVPN: yum -y install openvpn

Firewall changes now need to be made:

[root@Server1]# firewall-cmd --permanent --add-port=1194/tcp [root@Server1]# firewall-cmd --permanent --add-masquerade [root@Server1]# firewall-cmd --reload

Create Keys and Credentials on OPENVPN-SERVER use EasyRSA to create and sign the keys for the server and client. Install it with this:

yum -y install easy-rsa

Create a directory to hold the files you'll create:

mkdir /etc/openvpn/easy-rsa

and change to the directory...

cd /etc/openvpn/easy-rsa

To make things easier,append the EasyRSA executable folder to our current path:

PATH=$PATH:/usr/share/easy-rsa/3.0.7/

Initialize PKI:

easyrsa init-pki

Build the CA (remember the password you use, you can leave the common name as the default):

easyrsa build-ca

Generate a Diffie-Hellman key for forward secrecy:

easyrsa gen-dh

Now move on to the server credentials.

Create the server certificate:

easyrsa gen-req server nopass

Sign the server certificate:

easyrsa sign-req server server

Create the client certificate:

easyrsa gen-req client nopass

Sign the client certificate:

easyrsa sign-req client client

Type yes when prompted, and enter the same passwprd you entered for the server config

generate the TLS key:

cd /etc/openvpn openvpn --genkey --secret pfs.key

Configure the OpenVPN Server on OPENVPN-SERVER You'll need to create and edit /etc/openvpn/server.conf:

vim /etc/openvpn/server.conf

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

Enable & start openvpn via systemd systemctl enable openvpn@server.service systemctl start openvpn@server.service

Package up Keys and Certificates on OPENVPN-SERVER for Copying to CLIENT_UBUNTU

You'll need to package up the credentials you created, and copy them to CLIENT_UBUNTU, you can do this by creating the following shell script:

vim keys.sh

cd /etc/openvpn
mkdir -p server1/keys
cp pfs.key server1/keys
cp easy-rsa/pki/dh.pem server1/keys
cp easy-rsa/pki/ca.crt server1/keys
cp easy-rsa/pki/private/ca.key server1/keys
cp easy-rsa/pki/private/client.key server1/keys
cp easy-rsa/pki/issued/client.crt server1/keys
tar cvzf /root/keys.tgz OPENVPN-SERVER/```

Make it executable: chmod 700 keys.sh

And run it:

./keys.sh

Install OpenVPN on CLIENT_UBUNTU :

Just like on OPENVPN-SERVER, you'll need to install EPEL before you can install OpenVPN:

yum -y install epel-release
yum -y install openvpn

Copy and Install Keys from OPENVPN-SERVER to CLIENT_UBUNTU

Now you need to copy the keys you tar balled up on OPENVPN_SERVER over to CLIENT_UBUNTU

On CLIENT_UBUNTU :

scp root@<YOUR_SERVER_IP_HERE>:/root/keys.tgz ./

Configure the VPN client on CLIENT_UBUNTU

With the keys in place, you can now configure the client:

vim client.conf

client
dev tun
proto udp 
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
tls-auth server1/keys/pfs.key

Start & enable the Client:

systemctl start openvpn@client.service
systemctl enable openvpn@client.service

Add a Static Route on CLIENT_UBUNTU

In order to have CLIENT_UBUNTU traffic to node1 originate on the 10.8.0.0/24 network, you'll need to add a static route, so that the VPN tunnel is the interface that connects to that host:

ip route add 10.0.1.20 dev tun0

verify the entry using:

ip route show

