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

sudo apt install autoconf automake bats gcc \
    python3-sphinx python3-sphinx-rtd-theme rst2pdf fontconfig python3-yaml \
    latexmk texlive-latex-recommended texlive-latex-extra tex-gyre

sudo apt install -y gcc g++ make zip unzip php-fpm php-cli php-gd php-curl php-mbstring php-mysql php-json php-xml php-zip bsdmainutils ntp linuxdoc-tools linuxdoc-tools-text groff texlive-latex-recommended texlive-latex-extra texlive-fonts-recommended apache2 mysql-client mysql-server libapache2-mod-php libcgroup-dev
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
cd ~/domjudge/domserver/bin
./dj_setup_database genpass
cd ~/domjudge/domserver/etc
sudo vi dbpasswords.secret
```

You can edit this to make your own password to access the database. It will look something like this.

```
# Randomly generated on host ip-*-*-*-*, Thu Oct 26 12:03:44 UTC 2017
# Format: 'dummy:<db_host>:<db_name>:<user>:<password>'
dummy:localhost:domjudge:domjudge:_YourPasswordHere_
```

Resolve any dumb SQL errors

```
sudo rm -rf /etc/mysql
sudo rm -rf /var/lib/mysql*
sudo apt-get remove --purge mysql-server
sudo apt-get autoremove
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install mysql-server
```

Now to actually setup the database

```
cd ~/domjudge/domserver/bin
sudo ./dj_setup_database install
```

## Webserver

This is basically all from the docs and works by itself.

*Note: the php version will probably change in the future. Change to your version*

*Note: the ln and a2enmod might require sudo in front*

```
ln -s ~/domjudge/domserver/etc/apache.conf /etc/apache2/conf-available/domjudge.conf
ln -s ~/domjudge/domserver/etc/domjudge-fpm.conf /etc/php/8.1/fpm/pool.d/domjudge.conf
a2enmod proxy_fcgi setenvif rewrite
a2enconf php8.1-fpm domjudge
sudo systemctl reload php8.1-fpm
sudo systemctl reload apache2
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

*Note: this is probably because I installed domjudge in my home directory haha*

Now, you should be able to see the DOMJudge website with a login (`http://public_ip/domjudge`).

If you have any other errors, sorry :/

# Installing DOMJudge Judges

This following code will assume that the judges will be on another AWS EC2 instance. This is just good practice.

## Update Server

```
sudo apt update
sudo apt upgrade
```

## Dependencies

```
sudo apt install gcc g++ make cmake zip unzip debootstrap

sudo apt install acl zip unzip mariadb-server apache2 \
    php php-fpm php-gd php-cli php-intl php-mbstring php-mysql \
    php-curl php-json php-xml php-zip composer ntp

sudo apt install autoconf automake bats \
    python3-sphinx python3-sphinx-rtd-theme rst2pdf fontconfig python3-yaml \
    latexmk texlive-latex-recommended texlive-latex-extra tex-gyre

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
make dist
./configure --prefix=$HOME/domjudge
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

If you have a value of 1 or above, it is possible. It is really nice to run multiple judges on one machine. I do not have this ability (because of free tier) so I will not do the hyperthreading stuff.

## CGroups

Add `quiet cgroup_enable=memory swapaccount=1` to the end of `GRUB_CMDLINE_LINUX_DEFAULT`
```
sudo vi /etc/default/grub.d/50-cloudimg-settings.cfg 

/* In line 11 (probably), append the following */
GRUB_CMDLINE_LINUX_DEFAULT="~~existing stuff~~ quiet cgroup_enable=memory swapaccount=1”
```

> [!IMPORTANT]
> According to the documentation, it will have you edit the `/etc/default/grub1`. If you are running on your own hardware, that is fine but if you are using AWS, please do not do this. It will be overwritten by `/etc/default/grub.d/50-cloudimg-settings.cfg`. All of the parameters should be seen from `cat /proc/cmdline`. If you don't see everything that something is wrong.

> [!IMPORTANT]
> If you are using a system that uses cgroups v2 by default (like I ubuntu AWS), you need to add `systemd.unified_cgroup_hierarchy=0` to the `GRUB_CMDLINE_LINUX_DEFAULT`. This will force cgroups v1 which is what DOMJudge likes. This would look like:

```
sudo vi /etc/default/grub.d/50-cloudimg-settings.cfg 

/* In line 11 (probably), append the following */
GRUB_CMDLINE_LINUX_DEFAULT="~~existing stuff~~ quiet cgroup_enable=memory swapaccount=1 systemd.unified_cgroup_hierarchy=0”
```

Update change the changes. The reboot will take a little bit if your on AWS.

```
sudo update-grub
sudo reboot
```

Oh no, you've been kicked out of SSH! It's fine. Now you wait until the EC2 is done rebooting. This should take only about a minute.

## Connecting to the Server

You have to go into the actual server UI in order to make sure that your judgehost user and password are set. As the admin, go to the ip/domjudge/jury where you see a bunch of links to important places. Under Before Contest, go to the Users and then to the judgehost user. I was having trouble with this for a while and so I edit the user. You have to make sure that this user (judgehost) has the same username (judgehost) and password (should be automatically set but I did it manually and it worked) as the restapi.secret on the DOMJudge judgehost EC2 instance (found at .../domjudge/judgehost/etc/restapi.secreBefore Contest, go to the Users and then to the judgehost user. I was having trouble with this for a while and so I edit the user. You have to make sure that this user (judgehost) has the same username (judgehost) and password (should be automatically set but I did it manually and it worked) as the restapi.secret on the DOMJudge judgehost EC2 instance (found at .../domjudge/judgehost/etc/restapi.secret). The judge EC2 instance should now be able to talk to the DOMServer EC2 instance.

## Permissions

```
# In the etc file directory
cd ~/domjudge/judgehost/etc
sudo cp sudoers-domjudge /etc/sudoers.d/
```
## Creating the CGroups

```
# In the bin file directory
cd ~/domjudge/judgehost/bin
sudo groupadd domjudge-run
sudo useradd -g domjudge-run -M -s /bin/false domjudge-run
sudo systemctl enable create-cgroups --now
sudo ./dj_make_chroot
```

## Running the Judge

```
# In the bin file directory
cd ~/domjudge/judgehost/bin
./judgedaemon
```

## Current State

I am facing errors with the Judge not being able to complie any of the user submissions.

The internal error:

```
[Feb 21 03:51:17.656] judgedaemon[14619]:   💾 Fetching new executable 'compare/1' with hash 'cf0a04ed136ac59cb9ee3296a5fe9bba'.
[Feb 21 03:51:17.656] judgedaemon[14619]: API request GET judgehosts/get_files/compare/1
[Feb 21 03:51:17.684] judgedaemon[14619]: Building executable in /home/ubuntu/domjudge/judgehost/judgings/ip-172-31-30-105/endpoint-default/executable/compare/1/cf0a04ed136ac59cb9ee3296a5fe9bba, under 'build/'
[Feb 21 03:51:18.535] testcase_run.sh[15027]: starting '/home/ubuntu/domjudge/judgehost/lib/judge/testcase_run.sh', PID = 15027
[Feb 21 03:51:18.538] testcase_run.sh[15027]: arguments: '/home/ubuntu/domjudge/judgehost/judgings/ip-172-31-30-105/endpoint-default/testcase/1/b026324c6904b2a9cb4b88d6d61c81d1_59ca0efa9f5633cb0371bbc0355478d8.in' '/home/ubuntu/domjudge/judgehost/judgings/ip-172-31-30-105/endpoint-default/testcase/1/b026324c6904b2a9cb4b88d6d61c81d1_59ca0efa9f5633cb0371bbc0355478d8.out' '5:6' '/home/ubuntu/domjudge/judgehost/judgings/ip-172-31-30-105/endpoint-default/1/1/testcase00001'
[Feb 21 03:51:18.540] testcase_run.sh[15027]: optionals: '/home/ubuntu/domjudge/judgehost/judgings/ip-172-31-30-105/endpoint-default/executable/run/4/b5ea8d9ef4caee54508d7571c8206757/build/run' '/home/ubuntu/domjudge/judgehost/judgings/ip-172-31-30-105/endpoint-default/executable/compare/1/cf0a04ed136ac59cb9ee3296a5fe9bba/build/run' ''
[Feb 21 03:51:18.548] testcase_run.sh[15027]: setting up testing (chroot) environment
[Feb 21 03:51:18.565] testcase_run.sh[15027]: running program
[Feb 21 03:51:18.568] testcase_run.sh[15027]: runcheck: /home/ubuntu/domjudge/judgehost/judgings/ip-172-31-30-105/endpoint-default/executable/run/4/b5ea8d9ef4caee54508d7571c8206757/build/run testdata.in program.out sudo -n /home/ubuntu/domjudge/judgehost/bin/runguard -r /home/ubuntu/domjudge/judgehost/judgings/ip-172-31-30-105/endpoint-default/1/1/testcase00001/.. --nproc=64 --no-core --streamsize=8192 --user=domjudge-run --group=domjudge-run --walltime=5:6 --cputime=5:6 --memsize=2097152 --filesize=8192 --stderr=program.err --outmeta=program.meta -- /testcase00001/execdir/program
[Feb 21 03:51:18.649] testcase_run.sh[15027]: comparing output
[Feb 21 03:51:18.653] testcase_run.sh[15027]: starting compare script '/home/ubuntu/domjudge/judgehost/judgings/ip-172-31-30-105/endpoint-default/executable/compare/1/cf0a04ed136ac59cb9ee3296a5fe9bba/build/run'
[Feb 21 03:51:18.659] testcase_run.sh[15027]: runcheck: sudo -n /home/ubuntu/domjudge/judgehost/bin/runguard -u domjudge-run -g domjudge-run -m 2097152 -t 30 -c -f 2621440 -s 2621440 -M compare.meta -- /home/ubuntu/domjudge/judgehost/judgings/ip-172-31-30-105/endpoint-default/executable/compare/1/cf0a04ed136ac59cb9ee3296a5fe9bba/build/run testdata.in testdata.out feedback/
[Feb 21 03:51:18.714] testcase_run.sh[15027]: checking compare script exit-status: 255
[Feb 21 03:51:18.721] testcase_run.sh[15027]: Comparing failed with exitcode 255, compare script output:

---------- output validator stdout/stderr messages ----------
/home/ubuntu/domjudge/judgehost/bin/runguard: cannot start `/home/ubuntu/domjudge/judgehost/judgings/ip-172-31-30-105/endpoint-default/executable/compare/1/cf0a04ed136ac59cb9ee3296a5fe9bba/build/run': Permission denied
Try `/home/ubuntu/domjudge/judgehost/bin/runguard --help' for more information.
[Feb 21 03:51:18.730] testcase_run.sh[15027]: exiting with status '120'
[Feb 21 03:51:18.732] judgedaemon[14619]: comparing failed for compare script '1'
```
