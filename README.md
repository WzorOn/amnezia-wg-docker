## About The Project
Mikrotik compatible Docker image to run Amnezia WG tunnel on Mikrotik routers. As of now, support ARM64 boards(tested on hAP ax^3 7.15.3 (stable) as client and VPS amneziawg-linux-kernel-module@AlmaLinux9 as server)

## About The Project
This is a highly experimental attempt to run [Amnezia-WG](https://github.com/amnezia-vpn/amnezia-wg) 

### Prerequisites
Follow the [Mikrotik guidelines](https://help.mikrotik.com/docs/display/ROS/Container) to enable container support.
Install [Docker buildx](https://github.com/docker/buildx) subsystem, make and go.
```
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo apt install qemu-user-static
``` 

### Building Docker Image
Clone the repository
```
git clone https://github.com/WzorOn/amnezia-wg-docker.git
cd amnezia-wg-docker/
```
You may need(nope) to initialize submodules
```
git submodule add --force https://github.com/amnezia-vpn/amneziawg-go.git amneziawg-go
git submodule update --init --force --remote
```
To build a Docker container for the ARM64 v8 run
```
DOCKER_BUILDKIT=1  docker buildx build --no-cache --platform linux/arm64/v8 --output=type=docker --tag docker-awg:latest .
```
This command should cross-compile amnezia-wg locally and then build a docker image for ARM64 arch.
To export a generated image, use
```
docker save docker-awg:latest > docker-awg-arm8.tar
```
You will get the `docker-awg-arm8.tar` archive ready to upload to the Mikrotik router.

### Config awg
Generate a config for the client (mikrotik) via the [AmneziaVPN](https://github.com/amnezia-vpn/amnezia-client) application and rename it to awg0.conf
[Calculator WG AllowedIPs](https://www.procustodibus.com/blog/2021/03/wireguard-allowedips-calculator/) . Calculate WG AllowedIPs in the calculator. In the “Allowed IPs” field add 0.0.0.0.0/0, and in the “Disallowed IPs” field add private subnets (10.0.0.0/8, 100.64.0.0/10, 172.16.0.0/12, 192.168.0.0/16) and the IP of your VPS server, copy the calculation into the awg0.conf file in the AllowedIPs field.
Example of a config created by AmneziaVPN application and modified for our needs. 
```
[Interface]
Address = 10.8.1.3/32
DNS = 1.1.1.1, 1.0.0.1
PrivateKey = dmcc.../J1E=
Jc = 9
Jmin = 50
Jmax = 1000
S1 = 41
S2 = 50
H1 = 1959375798
H2 = 603385633
H3 = 1838903403
H4 = 1654724516

[Peer]
PublicKey = ngX6...wuVM=
PresharedKey = s543...8r39/E=
AllowedIPs = calculated_subnet
Endpoint = ip_your_vps:port_your_vps
PersistentKeepalive = 25
```
Сopy the docker archive and file configuration along the paths
```
usb1/docker/docker-awg-arm8.tar
usb1/docker/data/awg_conf/awg0.conf
```

### Mikrotik Configuration
Set up interface and IP address for the containers
```
/interface bridge add name=containers
/interface veth add address=172.19.0.2/24 gateway=172.19.0.1 gateway6="" name=veth1
/interface bridge port add bridge=containers interface=veth1
/ip address add address=172.19.0.1/24 interface=containers network=172.19.0.0
```
Add address list
```
/ip firewall address-list add address=2ip.ru list=manual-list
```
Add routing table
```
/routing table add disabled=no fib name=vpn-route-table
```
Add route mark
```
/ip firewall mangle add action=mark-routing chain=prerouting dst-address-list=manual-list new-routing-mark=vpn-route-table passthrough=yes
```
Add route for marked table
```
/ip route add disabled=no distance=2 dst-address=0.0.0.0/0 gateway=172.19.0.2 routing-table=vpn-route-table scope=30 suppress-hw-offload=no target-scope=10
```
Set up mount with the Wireguard configuration
```
/container mounts add name="awg_conf" src="/usb1/docker/data/awg_conf" dst="/etc/amnezia/amneziawg/"
/container/add cmd=/sbin/init hostname=amnezia interface=veth1 logging=yes mounts=awg_conf file=usb1/docker/docker-awg-arm8.tar
```
To start the container run
```
/container/start number=0
```
To get the container shell
```
/container/shell number=0
```