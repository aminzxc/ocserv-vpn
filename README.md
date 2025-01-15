### Install ocserv
```
sudo apt update
sudo apt install ocserv
```
### Create a minimal configuration file
```
sudo vim /etc/ocserv/ocserv.conf
```
### Add this basic configuration:
```
auth = "plain[passwd=/etc/ocserv/ocpasswd]"
tcp-port = 443
udp-port = 443
run-as-user = nobody
run-as-group = daemon

# Socket configuration
socket-file = /var/run/ocserv.socket
occtl-socket-file = /var/run/occtl.socket
use-occtl = true

device = vpns

# Certificate settings
server-cert = /etc/ocserv/ssl/server-cert.pem
server-key = /etc/ocserv/ssl/server-key.pem

# Network settings
ipv4-network = 192.168.1.0/24
dns = 8.8.8.8
dns = 8.8.4.4
route = default

cisco-client-compat = true
listen-host = 46.249.100.48

max-clients = 16
max-same-clients = 2


# Allow access to internal server network
route = 192.168.1.0/24
route = 10.0.0.0/8
route = 172.16.0.0/12
# Allow specific server ports
expose-ports = tcp:22,tcp:80,tcp:443,tcp:8070
```
### Create a VPN user
```
sudo bash -c 'echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf'
sudo sysctl -p
```
### Set up firewall rules
```
sudo ufw allow 443/tcp
sudo ufw allow 443/udp
```
### Configure NAT (replace eth0 with your network interface)
```
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo apt install iptables-persistent
sudo netfilter-persistent save
```
### create a simple self-signed certificate
```
# Create directory for certificates
sudo mkdir -p /etc/ocserv/ssl

# Generate private key
sudo certtool --generate-privkey --outfile /etc/ocserv/ssl/server-key.pem

# Create a simple template
sudo bash -c 'cat > /etc/ocserv/ssl/server.tmpl <<EOF
organization = "VPN"
cn = "VPN"
expiration_days = 3650
signing_key
encryption_key
tls_www_server
EOF'

# Generate self-signed certificate
sudo certtool --generate-self-signed \
--load-privkey /etc/ocserv/ssl/server-key.pem \
--template /etc/ocserv/ssl/server.tmpl \
--outfile /etc/ocserv/ssl/server-cert.pem
```
### To see the list of connected users in ocserv, you can use either of these commands
```
sudo occtl show users
```
### Show server status
```
sudo occtl show status
```
### disconnect a specific user
```
sudo occtl disconnect user [username]
```

