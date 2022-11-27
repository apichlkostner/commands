# Network

## Tap and bridge

### Tap
Networkmanager, permanent:
Create tap device with owner current user
```
nmcli connection add type tun ifname tap0 con-name tap0 mode tap owner `id -u`
```

### Bridge
Networkmanager, permanent:
Create bridge with manual ip address
```
nmcli connection add type bridge con-name br0 ifname br0 ip4 192.168.99.1/24
```

### Connect tap to bridge
Networkmanager, permanent:
```
nmcli connection modify tap0 master br0
```

* To enable internet access: connect bridge also to ethernet interface
* If internet is on Wifi, configure NAT between bridge and Wifi interface

### Connection up
```
nmcli connection up br0
nmcli connection up tap0
```

### dhcp and dns
```
sudo apt install dnsmasq
```
Configure in /etc/dnsmasq.conf
```
interface=br0       # bind to bridge
bind-interfaces     # only to bridge
dhcp-range=192.168.99.50,192.168.99.150,12h   # enable dhcp server for bridge
```

## Routing

### Enable routing:

Temporary:
```
sysctl net.ipv4.ip_forward=1
sysctl net.ipv6.conf.default.forwarding=1
sysctl net.ipv6.conf.all.forwarding=1
```
Permanent:

edit `/etc/sysctl.conf`
-> `net.ipv4.ip_forward=1`

### Routing from bridge to Wifi

Temporary:
```
sudo iptables -t nat -A POSTROUTING -o <name of wifi interface> -j MASQUERADE
sudo iptables -A FORWARD -i <name of bridge> -j ACCEPT
```

Permanent:
```
sudo apt install iptables-persistent
```
-> can save the current routes permanently

# Qemu

## Start qemu
```
qemu -enable-kvm -cpu host -machine q35 -device intel-iommu -hda ./archlinux.img -m 4096 -smp 4 -netdev tap,id=mytap0,ifname=tap0 -device virtio-net,netdev=mytap0,mac=52:54:00:12:34:56 -device virtio-vga-gl,edid=on,xres=1920,yres=1080 -display sdl,gl=on -monitor telnet:127.0.0.1:2099,server,nowait -audiodev pa,id=snd0 -device ich9-intel-hda -device hda-output,audiodev=snd0
```

# Mosquitto

* create passwords.txt with name:password
* process file sudo mosquitto_passwd -U ./passwords.txt
* configure password_file /etc/mosquitto/passwords.txt

## Configure server
/etc/mosquitto/mosquitto.conf   <- must be owned by user/group mosquitto
```
per_listener_settings true
listener 1883
password_file /etc/mosquitto/passwords.txt
listener 8883
certfile /etc/mosquitto/certs/server_crt.pem
keyfile /etc/mosquitto/certs/server_key.pem
cafile /etc/mosquitto/ca_certificates/ca.crt
password_file /etc/mosquitto/passwords.txt
```

Start mosquitto server/broker:
```
sudo mosquitto -c /etc/mosquitto/mosquitto.conf
```

## Client
Subscribe:
```
sudo mosquitto_sub -h <ip address> -t <topic> --cafile /etc/mosquitto/ca_certificates/ca.crt -u <name> -P <password>
```
Publish:
```
sudo mosquitto_sub -h 192.168.2.40 -t test --cafile /etc/mosquitto/ca_certificates/ca.crt -u <name> -P <password>
```


# Certificates

## Self signed
```
openssl genrsa -out ca.key 2048  # key for CA
openssl req -new -x509 -days 3650 -key ca.key -out ca.crt # root certificate
openssl genrsa -out server.key 2048 # generate key for server
openssl req -new -out server.csr -key server.key -addext subjectAltName=IP:192.168.2.40,DNS:hostname.local  # request for certificate for ip address
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 3650  # generate certificate
```

