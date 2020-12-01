# deploy-react-auth0
instructions to deploy a react app, [react-auth0-login](https://github.com/jimareed/react-auth0-login), to AWS.


## Setup

### 1. Provision AWS VM
- Login to AWS Management Console, select EC2 and select `Launch Instance`.
- Select `Amazon Linux` (default instance type) and select `Review and Launch`, then select `Launch`
- Note the Public IPv4 address

### 2. Configure Domain
- Start your Domain Manager (e.g., GoDaddy) and select the domain that you want to map to your VM, e.g., mysite.com. 
- Map your domain to the IPv4 address of the AWS VM.

### 3. Install Nginx & Docker
- login to your VM and install nginx & docker
```
ssh -i "ssh-key-file" mysite.com -l ec2-user
sudo -i
yum update
yum install docker
service docker start
amazon-linux-extras install nginx1.12
service nginx start
```
- test nginx:
```
curl localhost
```
This should return an NGINX html page

### 4. Set Inbound Security Rules
- from the AWS EC2 Dashboard, select your VM Instance and select the Security tab
- select your Security groups, select `Edit inbound rules`
- add the following rules:
  - HTTP, source Anywhere
  - HTTPS, source Anywhere

### 5. Test NGINX
- open your domain in a browser, should see the nginx default page (with a not secure warning)
<p  align="center">
    <img src="./images/nginx.png" alt="NGINX" width="75%" height="75%"/>
</p>

### 6. Build and run react-auth0-login locally
- Verify in your development environment before deploying to the VM. Follow the instructions in [react-auth0-login](https://github.com/jimareed/react-auth0-login) to configure the auth0 application and build and run the react app.

### 7. Install docker compose
- Install docker compose on the VM:
```
sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

### 8. Install react-auth0-login on VM
- copy the docker-compose.yaml file for react-auth0-login to your VM and run the following:
```
/usr/local/bin/docker-compose up -d
docker ps
curl localhost:3000
```
- `docker ps` should return an entry for the react-auth0-login app, the `curl` command should return the main HTML page.

### 10. Setup Cert

```
sudo -i
curl -O https://dl.eff.org/certbot-auto
chmod +x certbot-auto
mv certbot-auto /usr/local/bin/certbot-auto
```

```
service nginx stop
cd /usr/local/bin
vi certbot-auto
replace: elif [ -f /etc/redhat-release ]; then
with: elif [ -f /etc/redhat-release ] || grep 'cpe:.*:amazon_linux:2' /etc/os-release > /dev/null 2>&1; then
```
```
./certbot-auto certonly --standalone -d mysite.com
```
```
cd /etc/nginx
cp nginx.conf nginx.conf.org

vi nginx.conf
(uncomment TLS server section and replace the following lines)
        ssl_certificate "/etc/letsencrypt/live/mysite.com/fullchain.pem";
        ssl_certificate_key "/etc/letsencrypt/live/mysite.com/privkey.pem";
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers HIGH:SEED!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!RSAPSK:!aDH:!aECDH:!EDH-DSS-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA:!SRP;
        ssl_prefer_server_ciphers on;
```

### 11. Update nginx to serve the app
```
vi /etc/nginx/nginx.conf
(add the following in the TLS section)
        location / {
            proxy_pass http://localhost:3000;
        }
service nginx stop
service nginx start
curl localhost
```
- `curl` should return the same result as from step 8.
- opening https://mysite.com in a browser should behave the same as the localhost version in step 6

