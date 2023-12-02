# synapse-devops

# 1. Manual VM Setup with Debian 12

1. Download Debian 12 ISO: (https://www.debian.org/)
    
2. Download VirtualBox: https://www.virtualbox.org/ and sudo apt install virtualbox
    
3. launch virtualbox, and create a new Linux Debian (64-bit) VM called Debian_12. 
   Allocate memory of the machine with default settings (1GB RAM and 8GB Storage)
       
4. install debian OS onto your newly created VM from the downloaded debian.iso. Ensure that the new VM can be accessed into from host machine with ssh. 
   Or follow instructions from here https://dev.to/developertharun/easy-way-to-ssh-into-virtualbox-machine-any-os-just-x-steps-5d9i

5. turn off the newly created VM and change the network settings by adding an adapter and a port forwarding rule
    
6. in GRUB, start virtualbox machine in recovery mode to login as root and adduser pimwipa sudo so that pimwipa can do sudo, then exit root machine
    
7. then, turn on virtualbox again in normal mode (debian) as pimwipa@debian: install ssh server on the VM --> sudo apt update and sudo apt install openssh-server. in settings, go to network menu on the left, attach NAT network adapter, and set port forwarding host port 3022, guest port 22
    
8. now, back to the host machine ssh-copy-id pimwipa@localhost
or copy .ssh/key.pub (all of the key.pubs) and put in VM .ssh/authorized_keys and ssh -p 3022 pimwipa@127.0.0.1

I completed setting up the debian 12 VM and able to ssh into it without a password
Now I go on to install Ansible on my local machine. My local machine will act as Ansible controller to remote-control the debian VM that I have just created.

For Ansible to know where to get to my debian VM, I have to tell Ansible in etc/ansible/host the server name, host, and the ssh port that it has to go to

in etc/ansible/host

debian.synapse.com ansible_ssh_port=3022 ansible_host=localhost ansible_user=pimwipa

in .ssh/config

Host localhost
    Port 3022
    User pimwipa

Now my Ansible can ssh into my debian VM. I can check this with the command ansible all -m ping -u pimwipa



# 2.NGINX Web Server Installation and Configuration

1. in local machine, prepare the playbook file nginx.yml
2. run playbook with this command ansible-playbook -K -vvvv nginx.yml. When prompted for BECOME password:, input the local password 
3. in debian, Apache is the default webserver. To ensure that Nginx is in used instead of Apache, go to var/www/html in the VM and rename index.nginx-debian.html to be index.html
4. sudo lsof -i :80 should now show nginx

5. clear cache from the browser

6. in the VM, curl localhost or go to browser’s url box and input localhost, and it should show welcome to nginx! Page
7. config so that nginx can serve static website and accessible from host machine. In etc/nginx/sites-available/
sudo touch example-site and in that file, paste your configuration, for example;

server {

    listen 80;
    server_name example.com www.example.com;

    location / {
        root /var/www/html;
        index index.html;
    }

    # Additional NGINX configurations...
}


8. then create symlink to this file from etc/nginx/sites-enabled
sudo ln -s ../sites-available/example-site

9. sudo nginx -t  to test the syntax if everything s alright

10. sudo service nginx reload

Now I have nginx serving a static webpage on my debian VM.



# 3. MySQL Database Server Setup

1. in local machine, prepare the playbook file db.yml, and create_mysql_for_wordpress.yml

2. in etc/mysql/conf.d/mysql.cnf, add
[mysqld]
bind-address = 127.0.0.1

3. run playbook with this command ansible-playbook -K -vvvv db.yml

This will install mysql on the debian VM, enable remote login, and restart mysql service.

5. run another playbook ansible-playbook -K -vvvv create_mysql_for_wordpress.yml

This will install php dependencies for wordpress, login to mysql as root, and create new user, password, and a database for wordpress.
Now I can login to my debian VM and do sudo mysql -u WordPress -p, then enter the password I created for user WordPress, and I got to mariadb’s prompt.



# 4. WordPress Installation and Configuration

1. Go to wordpress.com and create a website of your own. Choose option for free hosting, launch the site, write down the domain name. Mine is pimwipa6.wordpress.com

2. in local machine, prepare the playbook file wordpress.yml. Include the wordpress domain name from item 1. in the playbook.

3. Run the playbook ansible-playbook -K -vvvv wordpress.yml.

This will download the wordpress tar.gz where in the package comes with wp-config-sample.php to be edited. The playbook won’t run successfully yet until all the files in the following items are prepared completely.

4. configure wordpress. In debian VM, at /var/www/html/wordpress/wp-config.php, include username, password, and database for wordpress created in the previous task. Get all the keys and credentials for wordpress API as instructed in the file for authentication.

5. setup nginx to track the wordpress website by adding a config file of the website for nginx

sudo nano /etc/nginx/sites-available/pimwipa6.wordpress.com
This is the same as task 2 item 7. Include the below in the file

server {
    listen 80;
    server_name pimwipa6.wordpress.com www.pimwipa6.wordpress.com;

    root /var/www/html/wordpress;
    index index.php index.html index.htm;

    error_log /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
}
      
      
6. now run the playbook again  ansible-playbook -K -vvvv wordpress.yml. This time it should have no failed item.
7. go to the wordpress site in the browser pimwipa6.wordpress.com
8. check the access log sudo nano /var/log/nginx/access.log
There should be some logs in there after visiting the site.

Now I completed WordPress installation and configuration process using Ansible, ensuring it utilizes nginx.




# 5. Documentation and Version Control

I have documented all the steps and configuration here.

Nonetheless, I have not yet completed the security hardening but it is still work in progress. Therefore, you can see some mysql credentials leak here to show you the development process of the assignment.

# 6. Bonus Challenge: Security Hardening (Optional)

What I could do more for security hardening
1. my debian VM
-disable direct login to ensure only ssh into the machine from Ansible controller is possible

2. nginx
-implement TLS/SSL

3. MySQL
-use ansible-vault for mysql credentials

4. Wordpress
-protect wp-config.php by restricting access to it
