# How to Deploy Django to AWS
## 1. Set Up AWS
1. Go to AWS console to create an instance, and connect with the instance using ssh client.
2. Create a Target Group that contains the instance
3. Create a Load Balancer with default security group. This is the "entry point" that users will be entering to access your instance. 
4. Set up RDS with a new security group.
## 2. Setup Project
```shell
sudo apt update
sudo apt install python3-pip python3-dev nginx
sudo apt install python3-virtualenv
```

Next, clone the git repository of the project you want to deploy.
```shell
git clone "{YOUR_PROJCET_REPO}"
```
Create a virtual environment using virtualenv
```shell
cd ~/{YOUR_PROJECT_DIRECTORY}
virtualenv venv
```
Activate virtual environment, and install all the dependencies.
```shell
source venv/bin/activate
pip install -r requirements.txt
```
Install Gunicorn as well.
```shell
pip install gunicorn
```

Set up the django project.
```shell
python3 manage.py makemigrations
python3 manage.py migrate
python3 manage.py collectstatic
```
**Make sure to allow the connection from the instance to the RDS server**\
RDS--> Database --> Security Group --> Edit Inbound Rules\
Type: PostgreSQL\
Source: Security Group for the Instance.

## 3. Configure Gunicorn and Nginx
### 3-1 Gunicorn
Create a system socket file for gunicorn
```shell
sudo vim /etc/systemd/system/gunicorn.socket
```
Paste the contents below and save the file
```
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock

[Install]
WantedBy=sockets.target
```
Next, create a service file for gunicorn
```shell
sudo vim /etc/systemd/system/gunicorn.service
```
Paste the contents below to the file
```
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/{PROJECT_NAME}
ExecStart=/home/ubuntu/{PROJECT_NAME}/venv/bin/gunicorn \
        --access-logfile - \
        --workers 3 \
        --bind unix:/run/gunicorn.sock \
        {PROJECT_CORE}.wsgi:application

[Install]
WantedBy=multi-user.target
```
Enable the gunicorn socket
```shell
sudo systemctl start gunicorn.socket
sudo systemctl enable gunicorn.socket
```
### 3-2 Nginx
Delete the default enabled site from nginx
```shell
cd /etc/nginx/sites-enabled/
sudo rm -f default
```
Create a config file for nginx 
```shell
sudo vim /etc/nginx/sites-available/{PROJECT_NAME}
```
Paste the contents below:
```
server{
    listen 80 default_server;
    server_name _;
    location = /favicon.ico {access_log off; log_not_found off; }
    location /static/ {
        root /home/ubuntu/{PROJECT_NAME};
    }
    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock
    }
}
```
Now activate the config file using the following command
```shell
sudo ln -s /etc/nginx/sites-available/bloc /etc/nginx/sites-enabled/
```
Load static files using the following command
```shell
sudo gpasswd -a www-data ubuntu
```
Restart nginx and allow the changes to take place
```shell
sudo systemctl restart nginx
sudo service gunicorn restart
sudo service nginx restart
```
**Whenever you modify your project, use the following commands to apply the changes**
```shell
sudo service gunicorn restart
sudo service nginx restart
```
Now, the server should be up and running. 
# 4. Connecting DNS to your Instance
1. Go to Route 53 from AWS
2. Create a hosted zone with your domain name
3. Create a record with no subdomain. Turn on alias, and apply to alias to application and classic load balancer, which is connected with your target group that has ec2 instance.
4. In your DNS provider, copy and paste all the nameservers from amazon.
# 5. Get SSL Certificate
1. Visit Amazon Certificate Manager
2. Enter your full domain, and create the certificate
3. After createing the certificate, create certificate record in the same hosted zone as the domain in route 53.
4. Allow HTTPS requests from port 443 in security group and load balancers.
