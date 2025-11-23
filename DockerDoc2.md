Start by making a DigitalOcean droplet. This one was made using the San Francisco data center with Ubuntu 24.04 (LTS) x64 as the operating system, the basic plan for the droplet size, the regular CPU with SSD as the disk type, and the $6 a month option. The droplet was given an ssh key and created. After launching the droplet with the console, install Docker. This was done referencing DigitalOcean's "How To Install and Use Docker on Ubuntu" page. Specifically the following commands were run: sudo apt update, sudo apt install apt-transport-https ca-certificates curl software-properties-common, curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -, sudo add-apt-repository
"deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable", apt-cache policy docker-ce, sudo apt install docker-ce. These commands install packages to allow for apt to use packages over HTTPS, adds a key for the official docker repository to the system, adds the Docker repository as a source apt can pull from, then installs Docker from there. After installing Docker, install Docker Compose with: sudo apt-get update, sudo apt-get install docker-compose-plugin. It turns out that Docker Compose was already installed  so this was unnecessary. 
The  next step is to install WireGuard with Docker Compose, which was done following the "Installing WireGuard UI using Docker Compose" page on Hetzner.com. First, run the following: sudo apt update, sudo apt upgrade. Then, create a directory for the .YML file with: sudo mkdir /opt/wg-ui. Then, create the .YML file with: sudo vim /opt/wg-ui/docker-compose.yml. This opens the new .YML file which is written with the following configuration:
services:
  wireguard:
    image: linuxserver/wireguard:v1.0.20210914-ls7
    container_name: wireguard
    cap_add:
      - NET_ADMIN
    volumes:
      - ./config:/config
    ports:
      - "5000:5000"
      - "51820:51820/udp"
    restart: unless-stopped
wireguard-ui:
    image: ngoduykhanh/wireguard-ui:latest
    container_name: wireguard-ui
    depends_on:
      - wireguard
    cap_add:
      - NET_ADMIN
    # use the network of the 'wireguard' service. this enables to show active clients in the status page
    network_mode: service:wireguard
    environment:
      - SENDGRID_API_KEY
      - EMAIL_FROM_ADDRESS
      - EMAIL_FROM_NAME
      - SESSION_SECRET
      - WGUI_USERNAME=admin
      - WGUI_PASSWORD=admin
      - WG_CONF_TEMPLATE
      - WGUI_MANAGE_START=true
      - WGUI_MANAGE_RESTART=true
    logging:
      driver: json-file
      options:
        max-size: 50m
    volumes:
      - ./db:/app/db
      - ./config:/etc/wireguard
    restart: unless-stopped
Then the configuration is slightly modified to change the port. The line '-"5000:5000"' can be changed to '-"80:5000"' to help prevent the port from being blocked on public Wi-Fi. 
Then, the container can be created with: sudo docker compose -f /opt/wg-ui/docker-compose.yml up -d. Since the port was changed, WireGuard can be brough up by entering http://<ip-addr> without a port at the end. Login to WireGuard with the username admin and password admin as these are the default. 
Then, to configure routing, the "Post Up Script" field should be populated with: iptables -A FORWARD -i %1 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth+ -j MASQUERADE. The second "Post Down Script" should be populated with: iptables -D FORWARD -i %1 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth+ -j MASQUERADE. Save the configuration then click "Apply Config". 
Next, navigate to the WireGuard Clients tab and create a new client with the name "myPhone". Download the WireGuard app from the app store on an iOS device. Then, from the WireGuard clients tab, locate the "myPhone" client and click on the QR code option. Scan this QR code with the WireGuard app on the iOS device and turn on the VPN. Using "ip.hetzner.com" shows the before and after for the ip address of this iOS device:
Before:

After:

To connect a Windows device, start by creating a new client as before, this time naming it "puter". Once it is created, from this same tab, select the Download option for the computer client, which will download the configuration file. If the option to "Apply Config" exists in the upper right, select it. Install the WireGuard desktop client for Windows and open it. Import the configuration file that was just downloaded. Right click on the "puter" listing in the bank on the left and select "Edit Selected Tunnel". If "Block untunneled traffic" is checked, uncheck it. Now the latest handshake should indicate that the connection is successful. Once again use "ip.hetzner.com" to show the before and after for the ip address of the Windows device:
Before:

After:
