## Steps and Commands

### Basic (MVP) getting started
1. Launch an EC2 micro ubuntu instance. 
   * Make sure to enable auto-assign public IP
   * Add or create a security group to allow inbound connections on ports 22 (ssh), 80 (html), 443 (https) and 5000
 (flask).
   * Be sure to request a private key and download it.
2. ssh into instance using private key (note the username is ubuntu and we assume the key was saved in ~/.ssh):
```
ssh -i ~/.ssh/<private key filename>.pem ubuntu@<public_ip>
```
3. Install and enable redis as a service: 
```
sudo apt -y update
sudo apt -y install redis-server
sudo systemctl start redis-server
sudo systemctl enable redis-server
```
4. Create snappass user and add to sudo group:
```
sudo adduser snappass
sudo usermod -aG sudo snappass
```
5. Clone the Qbiz snappass repo:
```
sudo su - snappass
git clone https://github.com/jaythomas-qbiz/qbiz-snappass.git
```
6. Install pip and requirements
```
sudo apt -y install python3-pip
pip3 install -r ~/qbiz-snappass/requirements.txt
```
7. Create `~/start_snappass.sh` script with the following content:
```
#!/usr/bin/env bash

cd /home/snappass/qbiz-snappass/snappass && python3 main.py
```
8. Set permissions:
```
chmod 755 ~/start_snappass.sh
```
7. Create `/etc/systemd/system/snappass.service` file with the following content:
```
[Unit]
Description=snappass daemon
After=network.target redis-server.service
Wants=redis-server.service
[Service]
EnvironmentFile=/etc/environment
User=snappass
Group=snappass
Type=simple
ExecStart=/home/snappass/start_snappass.sh
Restart=always
RestartSec=5s
[Install]
WantedBy=multi-user.target
```
8. Start and enable snappass service:
```
sudo systemctl start snappass.service
sudo systemctl enable snappass.service
```
9. Test snappass by going to `http://<public_ip>:5000` in a web browser



### Domain name
1. Allocate an [elastic IP address](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html) and associate it with the EC2 instance
2. Create a [Route 53 hosted zone](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/CreatingHostedZone.html) if you're not using an already-existing domain
3. Create an [A record](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-creating.html) in the hosted zone with the following settings:
    - Name: `<desired subdomain>`
    - Record Type: `A`
    - Value: `<elastic ip>`
4. Wait at least 60 seconds before testing the domain name in a web browser
5. Note that you can now ssh directly to your new subdomain
### NGINX and SSL
1. Install nginx:
```
sudo apt -y install nginx
```
2. Visit your domain name in a web browser (HTTP, without the `:5000`). You should see the default nginx page
3. Set up nginx to proxy requests to snappass.
   * ssh to the instance
   * create `/etc/nginx/sites-enabled/<subdomain>.<domain>` with the following content, replacing `subdomain.domain.com`
with your subdomain/domain name:
   ```
   server {
    listen 80;
    server_name subdomain.domain.com;

    # Redirect traffic to our locally-running flask app on port 5000
    location / {
        proxy_pass http://127.0.0.1:5000/;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Prefix /;
    }
   }
   ```
   * restart nginx:
   ```
   sudo systemctl restart nginx
   ```
   * visit your domain name in a web browser. You should see the snappass page

### SSL
(Domain name must be set up first)
1. Install certbot. Commands are below but see [here](https://certbot.eff.org/instructions?ws=nginx&os=ubuntufocal)
for more details
```
sudo snap install --classic certbot # snapd should already be installed; if not follow instructions from link above
sudo ln -s /snap/bin/certbot /usr/bin/certbot
sudo certbot --nginx
```
2. Answer questions when prompted. It should automatically detect most parameters such as domain name
3. Test that the site is now accessible via HTTPS by refreshing the page. You should see a lock symbol to the left
of the url

### If all went as expected... you're done!


