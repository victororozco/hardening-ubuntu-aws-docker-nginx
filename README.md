# Hardening Server (Ubuntu 18.04) with AWS - Docker - Nginx - Let's Encrypt

## AWS
- Install:
  ```bash
  sudo apt  install awscli 
  ```

- Configure AWS
  ```bash
  aws configure
  ```

- Login with AWS
  ```bash
  $(aws ecr get-login --no-include-email --region <region>)
  ```


## Docker

- Prerequisites
  ```bash
  sudo apt-get purge docker lxc-docker docker-engine docker.io
  sudo apt-get install  curl  apt-transport-https ca-certificates software-properties-common
  ```

- Setup Docker Repository
  ```bash
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add 
  sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
  ```

- Install Docker on Ubuntu

  ```bash
  sudo apt-get update
  sudo apt-get install docker-ce
  sudo systemctl status docker
  ```

- Added permission of Docker to user
  ```bash
  sudo usermod -aG docker ${USER}
  ```

- Configure AWS Credentials
  ```bash
  sudo vim  /etc/systemd/system/docker.service.d/aws-credentials.conf
  ```
  ```conf
  [Service]
  Environment="AWS_ACCESS_KEY_ID=<ACCESS_KEY_ID>"
  Environment="AWS_SECRET_ACCESS_KEY=<SECRET_ACCESS_KEY>"
  ```

- Execute:
  ```bash
  sudo systemctl daemon-reload
  sudo service docker restart 
  systemctl show --property=Environment docker # to see whether the env variables existed.
  ```

## Nginx

- Execute:
  ```bash
  sudo apt update
  sudo apt install nginx -y
  sudo vim /etc/nginx/sites-available/example.com
  ```

- Content example.com:
  ```nginx
  server {
          listen 80;
          listen [::]:80;

          root /var/www/example.com/html;
          index index.html index.htm index.nginx-debian.html;

          server_name example.com www.example.com;

          location / {
              proxy_set_header X-Real-IP       $remote_addr;
              proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
              proxy_set_header X-Forwarded-Proto https;
              proxy_cookie_path / "/; HTTPOnly; Secure";
              proxy_pass http://localhost:${PORT};

              proxy_connect_timeout 50000s;
              proxy_read_timeout 50000s;
              # try_files $uri $uri/ =404;
          }
  }
  ```

- Execute:
  ```bash
  sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/

  sudo vim /etc/nginx/nginx.conf
  ```

- Edit in nginx.conf and delete # in:
  ```nginx
  server_tokens off;

  server_names_hash_bucket_size 64;
  ```

## Nginx with Let's Encrypt

- Execute

  ```bash
  sudo add-apt-repository ppa:certbot/certbot
  sudo apt-get update
  sudo apt-get install python-certbot-nginx
  sudo vim /etc/nginx/sites-available/example.com
  ```

- Find the existing server_name line and replace the underscore, _, with your domain name:
  ```nginx
  . . .
  server_name example.com www.example.com;
  . . .
  ```

- Then, verify the syntax of your configuration edits.

  ```bash
  sudo nginx -t
  ```

- If it's Ok:
  ```bash
  sudo systemctl reload nginx
  ```

- Obtaining an SSL Certificate
  ```bash
  sudo certbot --nginx -d example.com -d www.example.com
  ```
  - Select option 2 **Redirect**

- Verifying Certbot Auto-Renewal
  ```bash
  sudo certbot renew --dry-run
  ```

## Others

### Enabling HTTP/2 Support
- Execute:
  ```bash
  sudo vim /etc/nginx/sites-available/example.com
  ```

- In the file, locate the listen variables associated with port 443:
  ```nginx
  ...
    listen [::]:443 ssl ipv6only=on; 
    listen 443 ssl; 
  ...
  ```

- Modify each listen directive to include http2:
  ```nginx
  ...
    listen [::]:443 ssl http2 ipv6only=on; 
    listen 443 ssl http2; 
  ...
  ```

- Removing Old and Insecure Cipher Suites
  - Locate the line that includes the options-ssl-nginx.conf file and comment it out:
    ```nginx
      # include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ```
  - Below that line, add this line to define the allowed ciphers:
    ```nginx
    ssl_ciphers EECDH+CHACHA20:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;
    ```

- Test nginx config:
  ```bash
  sudo nginx -t
  ```

- If it's OK:
  ```bash
  sudo systemctl reload nginx
  ```

