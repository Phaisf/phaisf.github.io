## Creating DigitalOcean Account
Using this [link](https://www.digitalocean.com/?refcode=d33d59113ab6&utm_campaign=Referral_Invite&utm_medium=Referral_Program&utm_source=CopyPaste), create a DigitalOcean account.
Go through steps to reach the dashboard and ENSURE you are under the **Free Trial Active** plan.

## Create Ubuntu Droplet
On the dashboard, click the **Spin up a Droplet**.
Choose the following options:
- Ubuntu 24.04 or 22.04
- Basic Droplet Size
- Regular Intel CPU and SSD choose 2nd cheapest option = $6/month
- Leave Datacenter as is
- Create a password or SSH key, password is easiest in this case, but SSH is much more secure (my password: @WacomCTH480PT)
- Hostname: ubuntu-s-1vcpu-1gb-sfo2-01-SYSADMIN
- Add Billing Address
Create the Droplet

## Installing Docker
https://docs.linuxserver.io/images/docker-wireguard/#usage
Upon creating the Droplet, click on the Droplet labeled whatever the hostname was named, in my case: **ubuntu-s-1vcpu-1gb-sfo2-01-SYSADMIN**.
After, click on the button to the right labeled **Console** to launch a console directly from DigitalOcean.
Using the same steps from [Project 2](https://phaisf.github.io/Docker_Project.html), follow the **Installation of Docker** section to install Docker and the respective packages.
- **NOTE:** Change the following parts to follow Ubuntu standards instead of Debian:
```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF

URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
```
Check to ensure Docker was successfully installed and running
```bash
sudo systemctl status docker
```
## Setting up WireGuard
Create a project directory for WireGuard and then a YAML file
```bash
mkdir wireguard-server
cd wireguard-server

nano docker-compose.yml
```
After using nano to modify the docker-compose.yml file, paste the following into the file
Used [WireGuard Documentation](https://docs.linuxserver.io/images/docker-wireguard/#docker-compose-recommended-click-here-for-more-info) to figure out how to make YAML file.
```bash
services:
  wireguard: 
    image: lscr.io/linuxserver/wireguard:latest
    container_name: wireguard

    cap_add:
      - NET_ADMIN
      - SYS_MODULE 

    environment:
      - PUID=0
      - PGID=0
      - TZ=America/Chicago
      - SERVERURL=157.230.167.241
      - SERVERPORT=80
      - PEERS=5
      - PEERDNS=auto
      - INTERNAL_SUBNET=10.13.13.0/24
      - PERSISTENTKEEPALIVE_PEERS=all

    volumes:
      - /lib/modules:/lib/modules
      - ./config:/config

    ports:
      - 80:51820/udp

    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
      - net.ipv4.ip_forward=1
    restart: unless-stopped
```
NOTE: Change the following
- SERVERURL to be your Droplet IPv4 address
- SERVERPORT to 80 to prevent being blocked on public WiFi
- PEERS=5, recommended base number but in this case, 2 is all we need as we are connecting with a phone and a laptop
- Under the **ports:** section, set up port forwarding from port 80 comms to port 51820 inside the Docker container by listing it as: 80:51820/udp
## Running WireGuard
Run `docker compose up -d` to start WireGuard
Next, look for the `./config` file and list the generated config file
Retrieve the files for two devices, a laptop and phone. `cat` the files and then copy and paste them later.
```bash
# Retrieve config for peer 1
cat ./config/peer1/peer1.conf

[Interface]
Address = 10.13.13.2
PrivateKey = uGOP0ibnGIrhzrDusGsHicIaHsoUtuTaTKTKum1xiE8=
ListenPort = 51820
DNS = 10.13.13.1

[Peer]
PublicKey = baKbq74TymFQ/1WK0F0G6gTHXrRrv6HZ6aXjHlmO1CY=
PresharedKey = lsrMWKWFTE/w/j86Jv5qO9UrSKjFIK8Gex/JbxGADJM=
Endpoint = 157.230.167.241:80
AllowedIPs = 0.0.0.0/0, ::/0

# Retrieve config for peer 2
cat ./config/peer2/peer2.conf

[Interface]
Address = 10.13.13.3
PrivateKey = mCD9VASZGI55l4HNWQ/AeGxJ7WpeAgQ+8Djkx+/89U4=
ListenPort = 51820
DNS = 10.13.13.1

[Peer]
PublicKey = baKbq74TymFQ/1WK0F0G6gTHXrRrv6HZ6aXjHlmO1CY=
PresharedKey = 1GXYEA06ETHJPvjFlxnpuzeiRsYohNAmcU9N4HuoPJQ=
Endpoint = 157.230.167.241:80
AllowedIPs = 0.0.0.0/0, ::/0
```
Create a QR code using `qrencode`, install it by using the following command and then generate a QR code of the `peer1.conf` file for a mobile phone to scan.
```bash
# Install qrencode
sudo apt update && sudo apt install qrencode -y

# Generate the QR code
qrencode -t ansiutf8 < ./config/peer1/peer1.conf
```
## Testing WireGuard
Visit https://ipleak.net/ to test IP before enabling VPN

### Mobile Device
In the WireGuard app, select **Add a tunnel** and then **Create from QR code**
After scanning the QR code, the tunnel will be created and can be activated.
Tunnel active
![[Pasted image 20251122114145.png]]

### Laptop
Install the WireGuard app using this [link](https://www.wireguard.com/install/) and choose the **Download Windows Installer**.
Using the information from the `peer2.conf` file, open **Notepad** and paste the info in. 
Save the file as **All files** and ensure the name is `peer2.conf` not `peer2.conf.txt`.
**Import tunnels from file** --> Choose the `peer2.conf` file

![[Pasted image 20251122113953.png]]