# Run Django project on AWS EC2 with NginX and Gunicorn (HTTP and HTTPS configuration tutorial)

This is a 10 step tutorial on how to run you Django application on an AWS EC2 instance.
In this walkthrough, we will learn how to clone a Django application and deploy it with NginX and Gunicorn and access the service via HTTP and HTTPS connections

## 1. Create an AWS account and an EC2 instance
For this step you can find a lot of tutorials online. You can just google it.
A few remarks in order to stay within the free tier:
* Only create instances of the type `t2.micro`. I recommend using an ubuntu server
* When creating a new instance click `next` in the form until you get to `storage`, free tier offers 30GB of SSD
* In the final step you will be asked to genarate a key file (.pem), save that and do not lose it. We will use that to connect to the instance

## 2. Create an generic EC2 git user 
In order to clone your repo in the instance it is best advised to create a separate EC2 user in your git service (GitHub, GitLab, ect.).
Add this generic user to your repo with `READ` access, this way it will only be used to read the latest changes.

## 3. Generate a public SSH key
First we need to connect to out instance. We use the file (.pem) that was generated for us when we created the instance.
Using the command `ssh -i <path to your .pem file> ubuntu@<your instance's IP/Hostname>` we have accessed the instance.
Now to generate the public key, use these commands:
* `$ssh-keygen -t rsa` - will generate a public key for us in ` ~/.ssh/id_rsa.pub`
* `$cat ~/.ssh/id_rsa.pub` - will display the contents of the public key file. Copy the contents

## 4. Add the public key to the generic EC2 user
Go to the git service provider, log in with the generic EC2 user. 
Go to settings, then over to SSH keys and add the public key there

Now the EC2 instance will be able to clone the repo over SSH.

## 5. Clone the project
I recomment creating some directories in the home folder like `mkdir -p git/<git service name>` and `cd git/<git service name>` there.
Now to clone the project with `git clone ssh@<repo location>`.

## 6. Test the service by exposing a custom port number
Assuming we will run our service on port `:8000`, we first have to go to the AWS Console, go to `Security Groups`, and edit the `inbound rules` of out instance. There we define a new rule with `custom TCP` and port number `8000` and save it.

Since we try a django project, we need to change the `ALLOWED_HOSTS` variable from `settings.py`. DON'T WORRY, after we will add NginX and Gunicorn, this will no longer be necessary.
* Change `ALLOWED_HOSTS = []` to `ALLOWED_HOSTS = ['<Instance IP>', '<Instance Hostname>']`

Run the django server using `python manage.py runserver`. This should start the service on the instance's localhost and on port 8000
Now if you open a browser and type `http://<instance host name>:8000`, you should see the Django welcome page. For this step you must add the port number in the URL.

## 7. Add NginX and Gunicorn in order to redirect all incomming requests on port 80 (HTTP) to your service on port 8000
In order to not add your service's port in the URL, we will add NginX and Gunicorn that will redirect all the traffic comming on port 80 (which is the default port for HTTP calls) to our running Django service on port 8000.

First of all we need to tell out AWS instance to allow incoming connections via HTTP (port 80). To do this, we head back to the AWS console, go to `Security Groups`, and here we edit the `inbound rules`. Add a new `HTTP` rule with `allow any`.

For NginX, we follow the next steps:
* `$sudo apt-get install nginx` - will install NginX
* Open file `$sudo nano /etc/nginx/nginx.conf` and change the first line to `user ubuntu;`
* Open file `$sudo nano /etc/nginx/sites-enabled/default`
* Edit the server object like this:
At the end of the file you will  find a commented out `server`. Uncomment that one and change to this: 
```
server {

  server_name 127.0.0.1 yourhost@example.com;
  access_log /var/log/nginx/domain-access.log;
  error_log  /var/log/nginx/nginx-error.log info;

  location / {
    proxy_pass_header Server;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    proxy_set_header X-Forwarded-For  $remote_addr;
    proxy_set_header X-Scheme $scheme;
    proxy_connect_timeout 10;
    proxy_read_timeout 10;

    # This line is important as it tells nginx to channel all requests to port 8000.
    # We will later run our wsgi application on this port using gunicorn.
    proxy_pass http://127.0.0.1:8000/;
  }
}
```
* `$sudo service nginx start` - will start the NginX server
Now if you open a browser and type `http://<instance host name>` you should see NginX welcome page

## 8. Add Gunicorn to run our WSGI application
In order to run our Django application we need a Python WSGI HTTP Server, and for this we will use Gunicorn
Follow the next steps to install and configure Gunicorn:
* `$pip install gunicorn` - this will install Gunicorn 
* `$cd <directory with wsgi.py>` - change the working directory where the wsgi.py file is.
* `$gunicorn wsgi -b 127.0.0.1:8000 --pid /tmp/gunicorn.pid --daemon` - this will bind Gunicorn to out Django server running on `127.0.0.1:8000` and `--deamon` will run the Gunicorn service in the background

If you are not interested in configuring HTTPS connections, then skip to step `Putting it all together`.

## 9. Configure HTTPS connections
For accepting HTTPS connections to our instance we need to perform some aditional steps.

### 9.1 Generate the SSL cetificats
First of all, create and change the working directory to
* `$mkdir -p /etc/nginx/certs && cd /etc/nginx/certs `

Then execute the following commands:
```
# generate CA key and certificate
$sudo openssl genrsa -des3 -out ca.key 4096
$sudo openssl req -new -x509 -days 365 -key ca.key -out ca.crt

# generate server key
# generate CSR (certificate sign request) to obtain certificate
$sudo openssl genrsa -des3 -out server.key 1024
$sudo openssl req -new -key server.key -out server.csr

# sign server CSR with CA certificate and key
$sudo openssl x509 -req -days 365 -in server.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out server.crt

# remove pass phase from server key
$sudo openssl rsa -in server.key -out temp.key
$sudo rm server.key
$mv temp.key server.key
```

### 9.2 Configure NginX with to listen on HTTPS requests
Open file `$sudo nano /etc/nginx/sites-enabled/default` and add a new server object like this at the end:
```
server {
    listen          443 ssl;
    server_name     yourhost@example.com;
    
    ssl_certificate     /etc/nginx/certs/server.crt;
    ssl_certificate_key /etc/nginx/certs/server.key;

    location / {
        proxy_pass http://127.0.0.1:8000/;
    }
}
```

### 9.3 Configure the EC2 instance to allow HTTPS connections
For this we need to tell out AWS instance to allow incoming connections via HTTPS (port 443). To achieve this, we head back to the AWS console, go to `Security Groups`, and here we edit the `inbound rules`. Add a new `HTTPS` rule with `allow any`.

### 9.4 Test the connections 
Run the command `$sudo service nginx restart` - will restart the NginX server
Now if you open a browser and type `https://<instance host name>` you should see NginX welcome page

## 10. Putting it all together
To fire everything up, we need to do the following:
* `python manage.py runserver` - this will start our Djnago application
* `gunicorn wsgi -b 127.0.0.1:8000 --pid /tmp/gunicorn.pid --daemon` - start Gunicorn, if not already started
* `sudo service nginx start` - start NginX if not already started

## 11. Conclusion
Now if you open a browser and type the following `http://<instance host name>` OR `https://<instance host name>` you should see the Django welcome page. :)

Note: Consider assigning an Elastic IP to your instance such that it will not change IP and hostname each time you restart it.
