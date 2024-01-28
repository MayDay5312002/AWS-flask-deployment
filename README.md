# Run Flask App on AWS EC2 Instance
Install Python Virtualenv
```bash
sudo apt-get update
sudo apt-get install python3-venv
```
Activate the new virtual environment in a new directory

Create directory
```bash
mkdir API
cd API
```
Create the virtual environment
```bash
python3 -m venv venv
```
Activate the virtual environment
```bash
source venv/bin/activate
```
Install Flask
```bash
pip install Flask
```
Create a Simple Flask API
```bash
sudo vi app.py
```
```bash
# Add this to app.py
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello_world():
	return 'Hello World!'

if __name__ == "__main__":
	app.run()
```
Verify if it works by running 
```bash
python app.py
```
Run Gunicorn WSGI server to serve the Flask Application
When you “run” flask, you are actually running Werkzeug’s development WSGI server, which forward requests from a web server.
Since Werkzeug is only for development, we have to use Gunicorn, which is a production-ready WSGI server, to serve our application.

Install Gunicorn using the below command:
```bash
pip install gunicorn
```
Run Gunicorn:
```bash
gunicorn -b 0.0.0.0:8000 app:app 
```
Gunicorn is running (Ctrl + C to exit gunicorn)!

Use systemd to manage Gunicorn
Systemd is a boot manager for Linux. We are using it to restart gunicorn if the EC2 restarts or reboots for some reason.
We create a <projectname>.service file in the /etc/systemd/system folder, and specify what would happen to gunicorn when the system reboots.
We will be adding 3 parts to systemd Unit file — Unit, Service, Install

Unit — This section is for description about the project and some dependencies
Service — To specify user/group we want to run this service after. Also some information about the executables and the commands.
Install — tells systemd at which moment during boot process this service should start.
With that said, create an unit file in the /etc/systemd/system directory
	
```bash
sudo nano /etc/systemd/system/API.service
```
Then add this into the file.
```bash
[Unit]
Description=Gunicorn instance for a simple hello world app
After=network.target
[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/API
ExecStart=/home/ubuntu/API/venv/bin/gunicorn -b localhost:8000 app:app
Restart=always
[Install]
WantedBy=multi-user.target
```
Then enable the service:
```bash
sudo systemctl daemon-reload
sudo systemctl start API
sudo systemctl enable API
```
Check if the app is running with 
```bash
curl localhost:8000
```
Run Nginx Webserver to accept and route request to Gunicorn
Finally, we set up Nginx as a reverse-proxy to accept the requests from the user and route it to gunicorn.

Install Nginx 
```bash
sudo apt-get nginx
```
Start the Nginx service and go to the Public IP address of your EC2 on the browser to see the default nginx landing page
```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```
Edit the default file in the sites-available folder.
```bash
sudo nano /etc/nginx/sites-available/default
```
Add the following code at the top of the file (below the default comments)
```bash
upstream flaskhelloworld {
    server 127.0.0.1:8000;
}
```
Add a proxy_pass to flaskhelloworld atlocation /
```bash
location / {
    proxy_pass http://flaskhelloworld;
}

location /endpoint {
    proxy_pass http://flaskhelloworld/route
}
```
Restart Nginx 
```bash
sudo systemctl restart nginx
```
Tada! Our application is up!

When making changes to API then you have to restart both API and nginx:


# HTTPS or SSL setup
### Nginx/HTTPS Configuration

#### About

HTTPS is when HTTP (hypertext transfer protocol); a communication protocol, is encrypted using TLS (transport layer security).

This means that if anyone is eavesdropping on the communication between a server and a client, they will eavesdrop on encrypted data that is hard to decipher and is therefore secure.

TLS is the more recent term which replaces the term SSL (secure socket layer) but we can refer to them as SSL/TLS.

The principle behind SSL is something you have probably come across before and it is public/private key pair.

Those 2 keys are mathematically related which means that if a message is encrypted with a public key, it can only be decrypted with the respective private key.

For SSL, we have a public key and a private key belonging to your domain and we have a certificate that validates  this public key belongs to your domain.

The certificate is signed by a trusted party called the certificate authority (CA).

When using Letsencrypt on your server to create a certificate, a public key and a private key are created. Only the public key is sent to the CA. The CA then gives you the SSL certificate to install on your server. Note that your browser has a list of trusted CAs that it can refer to.

#### Steps

You need to be on the server that has a DNS record for the domain that you want to create a digital certificate for.

Your server also has the firewall set up to allow traffic at the relevant ports.

You want to install the Letsencrypt client and the certbot Nginx plugin:

`sudo apt install certbot python3-certbot-nginx`

When you run certbot, the nginx config file gets updated automatically to include the SSL certificate path. It does that by identifying the block that contains the server_name directive.

`sudo certbot --nginx -d mydomain.com -d www.mydomain.com`

You will be prompted for an email address and other info.


The Docker-compose configuration:

```bash
nginx: 
 image : your_nginx_image/nginx:latest 
 ports : 
     - “80:80” 
     - “443:443”
 volumes: 
     - /path/to/cert:/etc/path/to/cert
```


The nginx config file /etc/nginx/sites-available/default

```bash
http {
        
    index index.html;

    server {
        server_name mydomain.com www.mydomain.com;

        root /home/user/build;

        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;

        ######### MOST IMPORTANT ##############
        listen 443 ssl; 
        ssl_certificate /etc/letsencrypt/live/mydomain.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/mydomain.com/privkey.pem;
        include /etc/letsencrypt/options-ssl-nginx.conf;
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;


        if ($host = www.mydomain.com) {
            return 301 https://$host$request_uri;
        }

        if ($host = mydomain.com) {
            return 301 https://$host$request_uri;
        }
    }

}

```

you can renew the certificate using the command below. You can schedule this on a monthly basis with Cron:

```bash
certbot renew
```

you can automatically renew the ssl by:
```bash
sudo crontab -l
```
you should see this line in /etc/cron.d/crontab
`0 */12 * * * root test -x /usr/bin/certbot -a \! -d /run/systemd/system && perl -e 'sleep int(rand(43200))' && certbot -q renew`
To see the config of crontab:
```bash
sudo cat /etc/cron.d/certbot
```

