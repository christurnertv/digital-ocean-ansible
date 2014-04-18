# Secure Web and DB Server Setup Guide 

This guide will walk you through all the steps to get your very own Web/DB Virtual Private Server (VPS) set up on Digital Ocean. We will be using Nginx as the web server and MySQL as the DB Server. It will be hard to find better value in a VPS than what we will build here.

When we are done, you will be able to host multiple sites on your server. You will also be able to push changes to your sites via Git for super easy deployment. Password authentication will be disabled via SSH (keys only). The firewall will only allow SSH, HTTP, and HTTPS.

Your sites will be stored in `/var/www` and your Git repositories will be in `/var/git`.

Note: This guide was written for Mac OSX.

## 1. Sign up for Digital Ocean

Please kindly follow my referral link below so I will get referral credit. :)

https://www.digitalocean.com/?refcode=e8387d479043

## 2. Generate SSH Keypair (if you do not already have one)

For secure authentication, we want to use key files instead of passwords. This requires you to have an SSH key setup on your machine and on the VPS. To see if you have this set up, type the following commands into your terminal:

```
cd ~/.ssh
ls -la
```

If you see files named `id_rsa` and `id_rsa.pub` you are good to go, if not, run the following commands (replace the sample email with your actual email address):

```
ssh-keygen -t rsa -C "name@example.com"
```

Accept the default file location when prompted by hitting enter.

When prompted for a passphrase, it is recommended that you enter one... but do not forget it. Every time you reboot and then attempt to use the key you will need to first enter the passphrase.

When key generation is complete, run the following command to make the key available to SSH.

```
ssh-add ~/.ssh/id_rsa
```

## 3. Add Public Key to Digital Ocean

By copying our public key to Digital Ocean, we will be able to use the private key on our Mac to authenticate to any virtual servers we create. Copy your public key to the clipboard by running the following command:

```
pbcopy < ~/.ssh/id_rsa.pub
```

Next, go to your Digital Ocean dashboard and click on "SSH Keys" in the left menu. Click the "Add SSH Key" button at the top of the page. Enter a name for the key like "Macbook Air" and then paste the key in the box below. When finished, click "Create SSH Key".

## 4. Create a Droplet

Follow the steps below to create your VPS.

- Click on the "CREATE" button to create a new Droplet.
- Enter a hostname for your Droplet. Something like "server1" is fine, but you can certainly be more creative.
- Select a size and region. The lowest size and any US region will work great.
- Select the Ubuntu 13.10 x64 image.
- Select the SSH Key you added above.
- Under the settings section, make sure that "Private Networking" is checked.- Click the "Create Droplet" button and wait a minute.

## 5. SSH into your VPS

Now that your VPS is all set up, SSH into it so that you can get it added to your SSH known hosts. Do so by running the following command (make sure to replace "ipaddress" with the IP of your server):

```
ssh root@ipaddress
```

If you are prompted to accept the identity of the remote server, type "yes" and hit enter. You should not have to enter a password if your SSH keys are set up per the instructions above. Once you verify that you can log in, you may type exit to quit.

## 6. Install Ansible and Dependencies

Ansible is an IT automation platform written in Python. We will be using it to install and configure software on our server from our Mac.

We will be installing Ansible from source as opposed to using `pip` since some of the modules are not loaded correctly when installed via `pip`. Run the following commands in your Mac OSX Terminal to install Ansible:

```
cd ~/Downloads
git clone https://github.com/ansible/ansible.git
cd ./ansible
sudo make install
```

Next, install dependencies as follows:

```
sudo easy_install pip
export CFLAGS=-Qunused-arguments
export CPPFLAGS=-Qunused-arguments
sudo ARCHFLAGS=-Wno-error=unused-command-line-argument-hard-error-in-future pip install paramiko PyYAML jinja2 markupsafe httplib2
```

## 7. Download, Configure, and Run Ansible Setup Script

First, clone this repository to your computer using the following command:

```
cd ~/Desktop
git clone https://github.com/cturner80/digital-ocean-ansible.git
```

From your Desktop, open the `hosts` file within the `digital-ocean-ansible` directory with your favorite editor. Inside, you will find a line that looks like this:

```
hostname    ansible_ssh_host=XXX.XXX.XXX.XXX    ansible_ssh_port=22    ansible_ssh_user=root    ansible_ssh_private_key_file=~/.ssh/id_rsa
``` 

Replace the `hostname` with whatever you entered when you created your Droplet. You will also need to change the value for the `ansible_ssh_host` to the IP address of your Droplet. When you are finished, save the file.

Now we are finally ready to get our server configured. It is as simple as running the following command (replace placeholders with real values):

```
cd ~/Desktop/digital-ocean-ansible
ansible-playbook -i hosts server-init.yml -e "hostname=server1" -e "domain=example.com" -e "admin_email=admin@example.com"
```

When the script is finished, your server will reboot. Once it reboots, you can start creating and deploying websites.

## 8. Create a New Site

We will walk through the setup of one website together. Subsequent sites can be configured the same way. Each site will have a separate php-fpm pool and nginx config file. It will also have a unique Git repository. All this is done automatically.

To create a new site, run the following commands (replace placeholders with real values):

```
cd ~/Desktop/digital-ocean-ansible
ansible-playbook -i hosts site-create.yml -e "hostname=server1" -e "domain=example.com"
```

When the command finishes running you will see a couple log statements that show the proper Git remote to add to your repository and also the command for how to push to the server. They will look something like this:

```
git remote add web ssh://root@example.com:22/var/git/example.com.git
git push web +master:refs/heads/master
```

You should use these commands in your local Git repository to be able to easily maintain the files on your server.

Note: Files are served from the `public` directory within the repo, not from the repo directory itself. Make sure you have a `public` directory in the root of your repo with you website files or this will not work.

## 9. Configure DNS and Access Site

To access your site on the web you will need to configure your domain name so that it points to your new Droplet. First, log into Digital Ocean and click on the "DNS" link in the left menu. Next, click "Add Domain". Enter your domain name and then use the drop down on the right to select your Droplet. The IP address will be auto-populated. Finally click "Create Domain". You will see a screen that shows your zone file. This will have a few entries like 
`ns1.digitalocean.com`. You will need these values for the next step.

Now the Digital Ocean knows how to route requests for our domain, we need to tell our domain registrar that we want to redirect requests to digital ocean. This varies from between domain registrars. The basic idea is that you need to tell your registrar to use Digital Ocean's name servers. Those the values we saw in the zone file in the previous step. Once you set up the name servers on your domain name now you just have to wait till the DNS propagates. This can take up to a day.

While you wait on DNS propagation, you can just add your domain entry to your local systems host file.

When all is said and done, you should be able to access your new website on your shiny new VPS from your web browser. Yay!

## Licensing

MIT licensed. Please keep my Digital Ocean referral link with the project. Thanks!

## Warranty

None, use at your own risk.

## Misc Notes

It is advisable to secure your MySQL installation by running the `mysql_secure_installation` command.

If you are running Laravel, do not for get to change the permissions on the `storage` directory and run composer.

```
cd /var/www/example.com
chmod -R g+w app/storage
composer install
```

If you want to use password authentication with Ansible in OSX, you will need to install sshpass as follows:

```
cd ~/Downloads
curl -O -L http://downloads.sourceforge.net/project/sshpass/sshpass/1.05/sshpass-1.05.tar.gz
tar xvzf sshpass-1.05.tar.gz
cd sshpass-1.05
./configure
sudo make install
```

Generate passwords to use in Ansible scripts as follows:

```
pip install passlib
python -c "from passlib.hash import sha512_crypt; print sha512_crypt.encrypt('<password>')"
```
