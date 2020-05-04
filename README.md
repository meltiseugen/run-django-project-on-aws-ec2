# run-django-project-on-aws-ec2

# 1. Create an AWS account and an EC2 instance
For this step you can find a lot of tutorials online. You can just google it.
A few remarks in order to stay within the free tier:
* Only create instances of the type `t2.micro`. I recommend using an ubuntu server
* When creating a new instance click `next` in the form until you get to `storage`, free tier offers 30GB of SSD
* In the final step you will be asked to genarate a key file (.pem), save that and do not lose it. We will use that to connect to the instance

# 2. Create an generic EC2 git user 
In order to clone your repo in the instance it is best advised to create a separate EC2 user in your git service (GitHub, GitLab, ect.).
Add this generic user to your repo with `READ` access, this way it will only be used to read the latest changes.

# 3. Generate a public SSH key
First we need to connect to out instance. We use the file (.pem) that was generated for us when we created the instance.
Using the command `ssh -i <path to your .pem file> ubuntu@<your instance's IP/Hostname>` we have accessed the instance.
Now to generate the public key, use these commands:
* `ssh-keygen -t rsa` - will generate a public key for us in ` ~/.ssh/id_rsa.pub`
* `cat ~/.ssh/id_rsa.pub` - will display the contents of the public key file. Copy the contents

# 4. Add the public key to the generic EC2 user
Go to the git service provider, log in with the generic EC2 user. 
Go to settings, then over to SSH keys and add the public key there

Now the EC2 instance will be able to clone the repo over SSH.

# 5. Clone the project
I recomment creating some directories in the home folder like `mkdir -p git/<git service name>` and `cd git/<git service name>` there.
Now to clone the project with `git clone ssh@<repo location>`.

# 6. Test the service by exposing a custom port number
Assuming we will run our service on port `:8000`, we first have to go to the AWS Console and edit the `inbound rules` of out instance.
There we define a new rule with `custom TCP` and port number `8000` and save it.

Since we try a django project, we need to change the `ALLOWED_HOSTS` variable from `settings.py`. DON'T WORRY, after we will add NginX and Gunicorn, this will no longer be necessary.
* Change `ALLOWED_HOSTS = []` to `ALLOWED_HOSTS = ['<Instance IP>', '<Instance Hostname>']`

Run the django server using `python manage.py runserver`. This should start the service on the instance's localhost and on port 8000
Now if you open a browser and type `http://<instance host name>:8000`, you should see the Django welcome page. For this step you must add the port number in the URL.

# 7. Add NginX and Gunicorn in order to redirect all incomming requests on port 80 (HTTP) to your service on port 8000
In order to not add your service's port in the URL, we will add NginX and Gunicorn that will redirect all the traffic comming on port 80 (which is the default port for HTTP calls) to our running Django service on port 8000.

For NginX, we follow the next steps:
* `sudo apt-get install nginx` - will install NginX
* Open file `sudo vim /etc/nginx/nginx.conf` and change the first line to `user ubuntu ubuntu;`
* Open file `/etc/nginx/sites-enabled/default`
