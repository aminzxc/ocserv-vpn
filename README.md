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
### create a user group configuration file
```
sudo vim  /etc/ocserv/group.conf
```
### Add these configurations
```
# Default group settings
[group:default]
# Bandwidth limits in bytes/sec (1MB = 1048576)
rx-per-sec = 1048576  # 1MB/s download
tx-per-sec = 1048576  # 1MB/s upload

# Traffic quota in bytes (1GB = 1073741824)
quota = 1073741824    # 1GB total traffic

# Connection limits
max-same-clients = 2  # Max simultaneous connections per user
session-timeout = 3600  # Session timeout in seconds (1 hour)
idle-timeout = 1200   # Idle timeout in seconds (20 minutes)

# Network restrictions
routes = default
no-routes = 192.168.1.0/24,192.168.0.0/24

# DNS servers
dns = 8.8.8.8
dns = 8.8.4.4
[group:premium]
rx-per-sec = 5242880  # 5MB/s download
tx-per-sec = 5242880  # 5MB/s upload
quota = 5368709120    # 5GB total traffic
max-same-clients = 5
session-timeout = 7200

[group:premium-plus]
# Bandwidth limits
rx-per-sec = 10485760  # 10MB/s download
tx-per-sec = 10485760  # 10MB/s upload

# Quota limits
quota = 107374182400  # 100GB total traffic

# Connection settings
max-same-clients = 5
session-timeout = 86400  # 24 hours
idle-timeout = 3600     # 1 hour

# Network settings
routes = default
no-routes = 192.168.1.0/24

# DNS settings
dns = 8.8.8.8
dns = 1.1.1.1

# Time restrictions
access-time = "mon-sun 00:00-23:59"

# Priority
net-priority = 1

```
### Additional Advanced Controls
```
# Time-based Access
[group:business-hours]
access-time = "mon-fri 09:00-17:00"

# IP-based Restrictions
[group:restricted]
allowed-ips = 192.168.1.0/24
denied-ips = 10.0.0.0/8

# Protocol Restrictions
[group:web-only]
restrict-user-to-ports = 80,443

# Custom Routes
[group:partial]
routes = 10.0.0.0/8,192.168.0.0/16
no-routes = 192.168.1.0/24

# Traffic Shaping

[group:throttled]
net-priority = 2
rx-per-sec = 512000  # 500KB/s download
tx-per-sec = 256000  # 250KB/s upload

# 
```
### Update your ocserv.conf to use groups
```
vim /etc/ocserv/ocserv.conf

# Enable group configurations
config-per-group = /etc/ocserv/group.conf

# Enable accounting
stats-report-time = 60

# Enable bandwidth and quota accounting
rate-limit-ms = 100
```
### Add users to specific groups
```
# Add a user to default group
sudo ocpasswd -c /etc/ocserv/ocpasswd username1

# Add a user to premium group
sudo ocpasswd -c /etc/ocserv/ocpasswd -g premium username2
```
### Show user statistics
```
sudo occtl show user username1
```
### Reset user quota
```
sudo occtl reset quota username1
```
### Access to the service
```
vpns0: flags=81<UP,POINTOPOINT,RUNNING>  mtu 1434
        inet 192.168.1.1  netmask 255.255.255.255  destination 192.168.1.161

services:
  nginx:
    image: nginx
    container_name: nginx
    ports:
      - "192.168.1.1:8070:80"  # Bind to VPN interface
```
