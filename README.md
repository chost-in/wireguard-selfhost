# WireGuard Docker Setup

This repository provides a detailed guide and configuration files for setting up a WireGuard VPN server using Docker with support for dual-stack IPv4 and IPv6.

## Features
- Docker-based WireGuard setup.
- Dual-stack IPv4 and IPv6 support.
- Configurable environment variables for easy setup.
- Automated subnet and peer configuration.

## Prerequisites
1. IPv6 enabled on the host server.
2. Basic understanding of networking and VPNs.

## Setup Instructions

### 1. This example uses Ubuntu. You can check installation instructions for other distributions in the [Docker Install Docs](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository).
Run the following commands to install Docker:
```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# Install Docker and Docker Compose
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add current user to Docker group
sudo usermod -aG docker $USER
```
**Note**: Log out and log back in to apply group changes.

### 2. Create Docker Network for IPv6
Run the following commands to create an IPv6-enabled Docker network:
```bash
# Create a Docker network with IPv6 support
# Replace the subnet below if you wish to use a different custom IPv6 subnet
docker network create \
  --ipv6 \
  --subnet fd7e:247e:2031::/64 \
  ipv6net

docker network inspect ip6vpn
```

### 3. Deploy WireGuard Using Docker Compose
1. Clone this repository:
   ```bash
   git clone https://github.com/chost-in/wireguard-selfhost/docker-compose.yaml
   cd wireguard-selfhost
   ```
 or
1. Create a folder with name `wireguard-selfhost` and create a nano file with name `docker-compose.yaml` in it.
2. Customize the `docker-compose.yaml` file as needed:
   ```yaml
   version: '3.8'
   services:
     wireguard:
       image: lscr.io/linuxserver/wireguard:latest
       container_name: wireguard
       cap_add:
         - NET_ADMIN
         - SYS_MODULE
       environment:
         # Replace the following environment variables as needed:
         - PUID=1001              # User ID on the host system
         - PGID=1001              # Group ID on the host system
         - TZ=Asia/Calcutta       # Timezone
         - SERVERURL=wg.tva.pw    # Replace with your server's domain or public IP
         - SERVERPORT=51820       # Port number for WireGuard, you can change if it is already in use 
         - PEERS=1                # Number of client peers to generate
         - PEERDNS=1.1.1.1, 2606:4700:4700::1111 # DNS servers (IPv4 and IPv6 using cloudflare you can change if necessary)
         - INTERNAL_SUBNET=10.13.13.0/24, fd7e:247e:2031::/64 # Internal subnets (IPv4 and IPv6 modify it as you need)
         - ALLOWEDIPS=0.0.0.0/0, ::/0 # Allowed IPs for client traffic
         - PERSISTENTKEEPALIVE_PEERS=
         - LOG_CONFS=true
       volumes:
         - /home/ubuntu/wireguard:/config
         - /lib/modules:/lib/modules
       ports:
         - 51820:51820/udp
       sysctls:
         - net.ipv4.conf.all.src_valid_mark=1
         - net.ipv6.conf.all.disable_ipv6=0
         - net.ipv6.conf.all.forwarding=1
         - net.ipv4.conf.all.forwarding=1
       restart: unless-stopped
       networks:
         - ip6vpn

   networks:
     ip6vpn:
       external: true
       enable_ipv6: true
       ipam:
         driver: default
         config:
           - subnet: "10.13.13.0/24" # Default IPv4 subnet
           - subnet: "fd7e:247e:2031::/64" # Replace with your custom IPv6 subnet
   ```
3. Start the container:
   ```bash
   docker compose up -d
   ```

### 4. Configure Peers
1. Navigate to the `peer1/` directory.
2. Open `peer1.conf` and add the necessary IPv6 subnet details:
   ```
   # By default, IPv4 subnet will be provisioned. Ensure IPv6 is added as follows:
   Address = 10.13.13.1, fd7e:247e:2031::2
   AllowedIPs = 10.13.13.2/32, fd7e:247e:2031::3/128
   ```
3. Check PostUP and PostDown is allowing IPV6, for reference only
   ```
   PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth+ -j MASQUERADE; ip6tables -A FORWARD -i %i -j ACCEPT; ip6tables -A FORWARD -o %i -j ACCEPT; ip6tables -t nat -A POSTROUTING -o eth+ -j MASQUERADE
   PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth+ -j MASQUERADE; ip6tables -D FORWARD -i %i -j ACCEPT; ip6tables -D FORWARD -o %i -j ACCEPT; ip6tables -t nat --D POSTROUTING --out-interface eth+
   ```
4. Restart the container:
   ```bash
   docker restart wireguard
   ```

## Testing
To test IPv6 connectivity from within the container, run:
```bash
# Test IPv6 connectivity by pinging a public IPv6 address
docker exec -it wireguard ping6 2001:4860:4860::8888
```

## Troubleshooting
- Ensure IPv6 is enabled on the host server.
- Check Docker network configurations with `docker network inspect`.
- Use logs for debugging:
  ```bash
  docker logs wireguard
  ```

## License
This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
