# Setup DOMJudge Server

This is a minimal install of the DOMJudge server. This will affect some of the choices made within this instruction set to try and minimize the cost of the server.

> [!IMPORTANT]
> Staying within the free tier is not possible because of the web traffic. It will cost a few cents in order to run the server for competitions but it will not be a large amount of money. Depending on the load, it will range from a dollar to a sandwitch from subway.

## Make The Account

Go through the process to make a new account. This account will need to have a payment method in order for it to work. Go through the entire process of verifying your account and then your almost set to go.

> [!NOTE]
> This will take some time. After completing the verification process, you will have to wait an hour or two (up to 24h according to AWS)

## Make The Instance

On the homepage in the first widget, click on the EC2 and this will bring you to the EC2 page. Create An Instance to host the actual DOMJudge. This instance is going to be Ubuntu preset with t2.micro (for that free tier).

> [!IMPORTANT]
> The actual EC2 will depend on your specific needs. This is a minimal installation that attemps to stay mostly within free tier (not entirely possible).

Create a simple RSA keypair (pem) and store it in a safe location on your computer (You will use this later).

Configuring the network settings. I have set allow SSH traffic from any IP (I know that is not good practice but this is convient). Also set the web traffic to true for HTTPS and HTTP for the actual website I think. I have set the storage to 10GB but I'm sure the default 8GB would work just as well (GP2). As long as it is within the free tier.

## Setup the Database

Now that the EC2 Instance has been created, we can now ssh into the server. Go to the actual EC2 instance in AWS probably under my EC2 instances or something. Here you can find the public IP.

SSH into the server to start installing

```
ssh -i /path/to/pemFile ubuntu@public_ip_address_of_EC2
```

## Update Server

> [!WARNING]
> The dependencies will not work correctly without an update to the system.

```
sudo apt update
sudo apt upgrade
```

## Dependencies

```
sudo apt install acl zip unzip mariadb-server apache2 \
    php php-fpm php-gd php-cli php-intl php-mbstring php-mysql \
    php-curl php-json php-xml php-zip composer ntp

sudo apt install autoconf automake bats \
    python3-sphinx python3-sphinx-rtd-theme rst2pdf fontconfig python3-yaml \
    latexmk texlive-latex-recommended texlive-latex-extra tex-gyre
```

Now to actually download all of the code for DOMJudge. For this install, I am using DOMJudge 8.2.2:

```
wget https://www.domjudge.org/releases/domjudge-8.2.2.tar.gz
```

Extract the downloaded release file:

```
tar -xf domjudge-8.2.2.tar.gz
```

## Bootstrap The Configuration File

Since the downloaded and extracted domjudge doesn't inherently have a `configure` script by default, you will have to manually build it.

```
cd domjudge-8.2.2
make dist
```

## Install DOMJudge Server

This will install the actualy DOMServer in the ~ directory. You can run the `configure` script without the prefix and I think it will install to `/opt/` but I'm not sure about that.

```
./configure --prefix=$HOME/domjudge
make domserver
sudo make install-domserver
```

## Database Configuration

Create the randomly generated password for the database. Inside the `domserver/bin`:

```
./dj_setup_database genpass
sudo vi /opt/domjudge/domserver/etc/dbpasswords.secret
```

You can edit this to make your own password to access the database. It will look something like this.

```
# Randomly generated on host ip-*-*-*-*, Thu Oct 26 12:03:44 UTC 2017
# Format: 'dummy:<db_host>:<db_name>:<user>:<password>'
dummy:localhost:domjudge:domjudge:_YourPasswordHere_
```

Now to actually setup the database

```
sudo ./dj_setup_database install
```

## Webserver

This is basically all from the docs and works by itself.

*Note: the php version will probably change in the future. Change to your version*

```
ln -s <DOMSERVER_INSTALL_PATH>/etc/apache.conf /etc/apache2/conf-available/domjudge.conf
ln -s <DOMSERVER_INSTALL_PATH>/etc/domjudge-fpm.conf /etc/php/8.1/fpm/pool.d/domjudge.conf
a2enmod proxy_fcgi setenvif rewrite
a2enconf php8.1-fpm domjudge
sudo systemctl reload php8.1-fpm
sudo systumctl reload apache2
```

Now your webserver should be working. Put in the public IP address to the server into the search bar (http not https). The public IP by itself should show a "It's working page" from apache. Now lookup: `http://public_ip/domjudge`. If you get an error page (Forbidden), first check the error logs:

```
sudo tail -f /var/log/apache2/error.log
```

I had a permission error and this is what worked for me.

```
# chmod +x /home/<USERNAME>
# sudo chmod +x /home/
# sudo chmod +x /home/ubuntu/
# sudo chmod +x /home/ubuntu/__everything_in_between_too__/public/
chmod +x /home/ubuntu
```

Now, you should be able to see the DOMJudge website with a login (`http://public_ip/domjudge`).

# Installing DOMJudge Judges

This following code will assume that the judges will be on another AWS EC2 instance. This is just good practice.

## Update Server

```
sudo apt update
sudo apt upgrade
```

## Dependencies

```
sudo apt install gcc g++ make cmake zip unzip debootstrap php-cli php-zip php-curl php-json procps openjdk-8-jre-headless openjdk-8-jdk libcgroup-dev -y

sudo apt install acl zip unzip mariadb-server apache2 \
    php php-fpm php-gd php-cli php-intl php-mbstring php-mysql \
    php-curl php-json php-xml php-zip composer ntp

sudo apt install autoconf automake bats \
    python3-sphinx python3-sphinx-rtd-theme rst2pdf fontconfig python3-yaml \
    latexmk texlive-latex-recommended texlive-latex-extra tex-gyre
```

```
# More packages
sudo apt install make pkg-config sudo debootstrap libcgroup-dev \
      php-cli php-curl php-json php-xml php-zip lsof procps

# Uninstall apport because it conflicts with judgehosts?
sudo apt remove apport
```


## Download DOMJudge 8.2.2

```
wget https://www.domjudge.org/releases/domjudge-8.2.2.tar.gz
tar -xf domjudge-8.2.2.tar.gz
```

```
# Go back to the ~/domjudge-8.2.2 to build the judgehosts
cd ~/domjudge-8.2.2
make judgehost
sudo make install-judgehost
```

## Hyperthreading?

Check for hyperthreading capabilities

```
lscpu | grep "Thread(s) per core"
# or
cat /sys/devices/system/cpu/smt/active
```

If you have a value of 1 or above, it is possible. It is really nice to run multiple judges on one machine. I do not have this ability so I will not do the hyperthreading stuff.

## CGroups

```
sudo vi /etc/default/grub.d/50-cloudimg-settings.cfg 

/* In line 11 (probably), append the following */
GRUB_CMDLINE_LINUX_DEFAULT="~~existing stuff~~ quiet cgroup_enable=memory swapaccount=1”
```

```
# Update changes; the reboot will take a little bit
sudo update-grub
sudo reboot
```

## Connecting to the Server

You have to go into the actual server UI in order to make sure that your judgehost user and password are set. As the admin, go to the ip/domjudge/jury where you see a bunch of links to important places. Under Before Contest, go to the Users and then to the judgehost user. I was having trouble with this for a while and so I edit the user. You have to make sure that this user (judgehost) has the same username (judgehost) and password (should be automatically set but I did it manually and it worked) as the restapi.secret on the DOMJudge judgehost EC2 instance (found at .../domjudge/judgehost/etc/restapi.secreBefore Contest, go to the Users and then to the judgehost user. I was having trouble with this for a while and so I edit the user. You have to make sure that this user (judgehost) has the same username (judgehost) and password (should be automatically set but I did it manually and it worked) as the restapi.secret on the DOMJudge judgehost EC2 instance (found at .../domjudge/judgehost/etc/restapi.secret). The judge EC2 instance should now be able to talk to the DOMServer EC2 instance.


