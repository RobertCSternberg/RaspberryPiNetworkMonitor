# Raspberry Pi Network Monitoring Guide
A full walkthrough guide to building a Raspberry Pi Network Monitor with Status Pages via Uptime Kuma and ARP Monitoring of Smart Home Devices.

## I. Raspberry Pi Set Up
#### 1. Install a Operating System on the Raspberry Pi
Install Raspberry Pi OS 64-Bit Lite (Headless Version) using Raspberry Pi Imager

_Alternatively Install Raspian 64-Bit Lite (Debian, Headless Version) using NOOBS_

#### 2. After boot, setup username and password
In this guide we recommend using the 1st letter of your first name followed by your last name.  

For John Doe, your username would be jdoe. 

_After setup, the screen will go black.  At this point you should restart manually._

#### 3. Configure the Raspberry Pi Settings
Use `sudo raspi-config`
  - Set Network SSID & Password
  - Set Hostname
  - Set Login Method
  - Set Timezone
  - Update the Raspberry Pi Configurator

#### 4. Update and Upgrade the Raspberry Pi Packages
```
sudo apt update && sudo apt upgrade -y
```

#### 5. Reboot the Raspberry Pi
```
sudo reboot
```

## II. Installing and Configuring SSH
*Special Thanks to linuxHint: https://linuxhint.com/enable-ssh-server-debian/*

#### 1. Install Open SSH
##### a. Install the Open SSH Server
```
sudo apt install openssh-server
```
##### b. Start the SSH Service
```
sudo systemctl start ssh
```
***Optional: At this point, you can connect over SSH and complete the remainder of the guide.***
<br />
<br />
 
##### c. Enable SSH on Boot
```
sudo systemctl enable ssh
```
##### d. Validate Results
```
sudo systemctl status ssh
```

#### 2. Configure SSH (Optional)
##### a. Edit the SSH Config File
```
sudo nano /etc/ssh/sshd_config
```
 - Uncomment the PORT line set and the port to 2222
*Once you’re done, press <Ctrl> + X followed by Y and <Enter> to save the sshd_config file.*

##### b. Restart the SSH Service
```
sudo systemctl restart ssh
```
 - New command to connect is ssh <username>@<ip-addr> -p <port-number>
 - Use hostname -I to get the IP address.

***Optional: At this point, you will need to reconnect if you are currently connected over SSH.***


## III. Install Docker

#### 1. Install from the latest image. 
```
curl -sSl https://get.docker.com | sh
```

#### 2. Add your user to the `docker` group. 
```
sudo usermod -aG docker [Your Username]
```

## IV. Install Uptime Kuma

#### 1. Install from the latest image. 
```
sudo docker run -d --restart=always -p 3001:3001 -v uptime-kuma:/app/data --name uptime-kuma louislam/uptime-kuma:1

```

#### 2. Validate Results
```
sudo docker ps -a
```

## V. Uptime Kuma Login Creation

#### 1. Browse to Uptime Kuma

Access the login page at `[Your Raspberry Pi IP Address]:3001'

Example: `192.168.1.2:3001' *You can type this directly into your browser*

***Remember to configure 2 Factor Authentication***

## VI. Using Arp Web Server for Uptime of Devices that Do Not Allow Ping
### Create Arp Web Server

#### 1. Make a Directory and Enter it. 
```
mkdir networkscanner && cd networkscanner

```

#### 2. Create and Edit Dockerfile using `nano Dockerfile`

```
FROM nginx:alpine
RUN apk update && apk add --no-cache arp-scan
COPY run_scan.sh /run_scan.sh
RUN chmod +x /run_scan.sh
COPY nginx.conf /etc/nginx/nginx.conf
CMD /bin/sh -c "/run_scan.sh & nginx -g 'daemon off;'"
```
*Note: To save after using nano, press ctrl + x, Y, then enter.*

#### 3. Create and Edit the Scanning Script using `nano run_scan.sh`

```
#!/bin/sh
while true
do
  echo "<html><head><title>Arp Scan Results</title></head><body><h1>Arp Scan Results</h1>" > /tmp/index.html
  echo "<p>Last updated: $(TZ='America/Chicago' date '+%B %d, %Y %I:%M:%S %p') Central Time US</p>" >> /tmp/index.html
  echo "<pre>" >> /tmp/index.html

  # Call the curl command here. Replace the URL with the one you want to call from Uptime Kuma Push Monitor
  /usr/bin/curl "https://google.com" > /dev/null 2>&1

  # Replace the arp-scan address with your network address and subnet mask.  Most commonly 192.168.1.0/24
  arp-scan 192.168.1.0/24 >> /tmp/index.html 2>> /tmp/arp-scan-errors.log
  if [ $? -ne 0 ]; then
    echo "Arp-scan failed. See /tmp/arp-scan-errors.log for details." >> /tmp/index.html
  fi
  echo "</pre></body></html>" >> /tmp/index.html
  mv /tmp/index.html /usr/share/nginx/html/index.html
  cp /usr/share/nginx/html/index.html /usr/share/nginx/html/index-copy.html
  sleep 30
done
```
*Note: To save after using nano, press ctrl + x, Y, then enter.*

#### 4. Create and Edit the Webserver Configuration using `nano nginx.conf`

```
user nginx;
worker_processes 1;
events {
    worker_connections 1024;
}
http {
    server {
        listen 80;
        location / {
            root /usr/share/nginx/html;
            index index.html;
            try_files $uri $uri/ /index.html;
        }
    }
}
```
*Note: To save after using nano, press ctrl + x, Y, then enter.*

### Build and Run Docker Image

#### 1. Build the docker image from the files we just created. 
```
docker build -t networkscanner .
```

#### 2. Run the docker image and start the container, set the container to auto-restart, allow connections to the root network, allow the docker user to run the processes related to the arp scan. 
```
docker run --restart=always --network=host -d --cap-add=NET_RAW --cap-add=NET_ADMIN networkscanner
```

#### 3. Access Arp Web Server to Test Results
Access the ARP web server by visiting `http://[pi ip address]/` or test headless with `curl http://[pi ip address]/`



## VII. Scheduling Push Notification Tasks with Cron to allow for Raspberry Pi Uptime Monitoring

#### 1. Cron Job to Push Notify the Monitor Every Minute
```
* * * * * /usr/bin/curl "[PUSH URL]" > /dev/null 2>&1
```

#### 2. Cron Job to Push Notify the Monitor at Startup
```
@reboot /usr/bin/curl "[PUSH URL]" > /dev/null 2>&1
```





# Helpful Commands

### Power and Configuration
#### Enter Raspberry Pi Config
`sudo raspi-config`

#### Shutdown Raspberry Pi Safely
`sudo shutdown -h now`

#### Reboot Raspberry Pi Safely
`sudo Reboot`

### Check what port would be good for monitoring a device. 
#### Install Nmap for Open Ports Monitoring
```
sudo apt-get update
sudo apt-get install nmap
```

#### Use Nmap for Scanning

 - Top 1000 ports
```nmap [ip address to scan]```

 - All ports
```nmap -p- [ip address to scan]```

 - Check all ports even if ping is blocked
```nmap -p- -Pn [ip address to scan]```

 - Advanced Scan of all ports (half-handshake, skip ping, all ports)
```nmap -sS -Pn -p- [ip address to scan]```

### Port Management

 - Check which service is using port `80` using `sudo netstat -tuln | grep :80` or `sudo ss -tuln | grep :80`

 - If nginx service is using port '80', stop the service using `sudo systemctl stop nginx` and prevent nginx from starting up when the system reboots `sudo systemctl disable nginx`

### Docker Container Management

 - Check all docker containers using `sudo docker ps -a` or just check running docker containers using `sudo docker ps`

 - Remove stopped docker containers using `sudo docker container prune`

 - Start, Stop, or Remove a docker container using `sudo docker start [container id]`, `sudo docker stop [container id]`, and `sudo docker rm [container id]`

 - To enter into a running Docker container to test commands use  `docker exec -it [container id] /bin/sh `
