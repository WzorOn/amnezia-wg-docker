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
```
git clone https://github.com/anettoph/amnezia-wg-docker.git
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

### Running locally

Just run `docker compose up`
Make sure to create a `wg` folder with the `awg0.conf` file.
Example server `awg0.conf`:
```
[Interface]
Address = 10.8.0.1/24
PrivateKey = KF..Uw=
ListenPort = 51820
MTU = 1420
Jc = 1 ≤ Jc ≤ 128; recommended range is from 3 to 10 inclusive
Jmin = Jmin < Jmax; recommended value is 50
Jmax = Jmin < Jmax ≤ 1280; recommended value is 1000
S1 = S1 < 1280; S1 + 56 ≠ S2; recommended range is from 15 to 150 inclusive 
S2 = S2 < 1280; recommended range is from 15 to 150 inclusive
H1 = H1/H2/H3/H4 — must be unique among each other; recommended range is from 5 to 2147483647 inclusive
H2 = H1/H2/H3/H4 — must be unique among each other; recommended range is from 5 to 2147483647 inclusive
H3 = H1/H2/H3/H4 — must be unique among each other; recommended range is from 5 to 2147483647 inclusive
H4 = H1/H2/H3/H4 — must be unique among each other; recommended range is from 5 to 2147483647 inclusive

[Peer]
PublicKey = Fz..Dk=
PresharedKey = qa..sY=
AllowedIPs = 10.8.0.3/32, 172.17.0.1/32

```

### Mikrotik Configuration

Set up interface and IP address for the containers
```
/interface bridge add name=containers
/interface veth add address=172.17.0.2/24 gateway=172.17.0.1 gateway6="" name=veth1
/interface bridge port add bridge=containers interface=veth1
/ip address add address=172.17.0.1/24 interface=containers network=172.17.0.0
```
Add address list
```
/ip firewall address-list
add address=2ip.ru list=manual-list
```
Add route mark
```
/ip firewall mangle
add action=mark-routing chain=prerouting dst-address-list=manual-list new-routing-mark=vpn-route-table passthrough=yes
```

Add routing table
```
/routing table
add disabled=no fib name=vpn-route-table
```

Add route for marked table
```
/ip route
add disabled=no distance=2 dst-address=0.0.0.0/0 gateway=172.17.0.2 routing-table=vpn-route-table scope=30 suppress-hw-offload=no target-scope=10
```

Set up mount with the Wireguard configuration
```
/container mounts add name="awg_conf" src="/usb1/docker/data/awg_conf" dst="/etc/amnezia/amneziawg/" 
/container/add cmd=/sbin/init hostname=amnezia interface=veth1 logging=yes mounts=awg_conf file=docker-awg-arm8.tar
```

To start the container run
```
/container/start number=0
```

To get the container shell
```
/container/shell number=0
/container/shell number=1
```