# Before starting make sure virtualization is enabled on the host
# and make sure if your installing ubuntu as the main OS then must disable secure boot

# add a new user 

  sudo adduser cuckoo
# set password and other configurations

# adding new user to group sudo
sudo usermod -aG sudo cuckoo

# Initial Requirements
sudo apt-get -y install python python3-pip python-dev virtualenv python-setuptools curl git libffi-dev libssl-dev libjpeg-dev zlib1g-dev swig

# For web Interface
sudo apt-get -y install mongodb

# For database
sudo apt-get -y install postgresql postgresql-contrib libpq-dev

# pydeep
sudo apt-get -y install libfuzzy-dev ssdeep
sudo ldconfig
git clone https://github.com/kbandla/pydeep.git
cd pydeep
python setup.py build
python setup.py test
sudo python setup.py install
cd ..
rm -r pydeep

# install python2 and pip2
curl https://bootstrap.pypa.io/pip/2.7/get-pip.py --output get-pip.py
sudo python2 get-pip.py

# Installing valatility
sudo python2 -m pip install -U distorm3 yara pycrypto pillow openpyxl ujson pytz ipython capstone
sudo ln -s /usr/local/lib/python2.7/dist-packages/usr/lib/libyara.so /usr/lib/libyara.so
sudo python2 -m pip install -U git+https://github.com/volatilityfoundation/volatility.git

# Virtualbox
sudo apt-get -y install virtualbox

# adding user cuckoo to group vboxusers
sudo usermod -a -G vboxusers cuckoo

# tcpdump
sudo apt-get -y install tcpdump apparmor-utils
sudo aa-disable /usr/sbin/tcpdump

# adding user cuckoo to group pcap
sudo groupadd pcap
sudo usermod -a -G pcap cuckoo
sudo chgrp pcap /usr/sbin/tcpdump
sudo setcap cap_net_raw,cap_net_admin=eip /usr/sbin/tcpdump

# Verfication of last Command
getcap /usr/sbin/tcpdump
# /usr/sbin/tcpdump = cap_net_admin,cap_net_raw+eip

# logout and login as new user

# creating virtualenv and installing cuckoo and vmcloak
virtualenv -p python2.7 env
source env/bin/activate
sudo pip2 install cuckoo vmcloak

# Starting cuckoo to setup default configuration
cuckoo init
cuckoo community

# Installing Guest OS for cuckoo
wget https://cuckoo.sh/win7ultimate.iso
# in case of certificate error
wget https://cuckoo.sh/win7ultimate.iso --no-check-certificate

sudo mkdir /mnt/win7
sudo mount -o ro,loop win7ultimate.iso /mnt/win7

# creating network for virtual machines (in virtual environment)
vmcloak-vboxnet0

# Network Configuration
# replace eth0 with ethernet adaptor
sudo sysctl -w net.ipv4.conf.vboxnet0.forwarding=1
sudo sysctl -w net.ipv4.conf.eth0.forwarding=1

# set routes for vboxnet0
sudo iptables -t nat -A POSTROUTING -o eth0 -s 192.168.56.0/24 -j MASQUERADE
sudo iptables -P FORWARD DROP
sudo iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -s 192.168.56.0/24 -j ACCEPT
sudo iptables -A FORWARD -s 192.168.56.0/24 -d 192.168.56.0/24 -j ACCEPT
sudo iptables -A FORWARD -j LOG

# write a script for above commands and run that script everytime before starting cuckoo

# Installing win7
vmcloak init --verbose --win7x64 win7x64base --cpus 2 --ramsize 2048

# cloning win7 (so that we can create more copies in future using base version)
vmcloak clone win7x64base win7x64cuckoo

# VMCloak supports the installation of multiple software packages. A full list of supported packages and versions can be listed:
vmcloak list deps

# it's better to specify versions with packages
vmcloak install win7x64cuckoo pillow python27 chrome chrome.version=latest adobepdf adobepdf.version=11.0.19

# A software package can be installed with the following syntax: vmcloak install <image name> <package>. A specific version or a serialkey can be provided by adding: package.version=X or package.serialkey=X. If no version is selected, the default version will be picked. We will be installing some basic software packages:
vmcloak install win7x64cuckoo adobepdf pillow dotnet ie11 java flash vcredist vcredist.version=2015u3 wallpaper

# Optional step: Installing a Microsoft Office version so that Office document can be analyzed. Office 2007 is most likely to work, some builds of higher versions of Office sometimes cause issues with the Cuckoo MonitorCuckoo Monitor:
vmcloak install win7x64cuckoo office office.version=2007 office.isopath=/path/to/office2007.iso office.serialkey=XXXXX-XXXXX-XXXXX-XXXXX-XXXXX

# VMCloak will register a VirtualBox VM for each snapshot created. After snapshotting, it is no longer possible to change the image. The syntax of the snapshot command is: vmcloak snapshot <options> <image name> <vmname> <ip to use>
# Using the --count parameter, we can create multiple snapshots at once. Let’s create four:
vmcloak snapshot --count 4 win7x64cuckoo 192.168.56.101

# if encounter multi-attach disk vm error then fix it manually

# The above command will create VMs win7x64cuckoo1-4 with IPs 192.168.56.101-104.
# After VMCloak is finished, the VMs can be listed using:
vmcloak list vms

# Adding VMs
while read -r vm ip; do cuckoo machine --add $vm $ip; done < <(vmcloak list vms)

# Removing cuckoo1 from virtualbox.conf
# remove all the content from [cuckoo1] to osprofile = and cuckoo1 from list of vms

# To configure iproute2 with eth0 we’re going to open the /etc/iproute2/rt_tables file and roll a random number that is not yet present in this file with your dice of choice and use it to craft a new line at the end of the file e.g; "400	eth0"

# configuring Per Network Analysis
# For Internet Routing
# open routing.conf and write eth0 in front of internet

# Postgres as DBMS
sudo pip install psycopg2

sudo -u postgres psql
CREATE DATABASE cuckoo;
CREATE USER cuckoo WITH ENCRYPTED PASSWORD 'password';
GRANT ALL PRIVILEGES ON DATABASE cuckoo TO cuckoo;
\q

# After that, we have to tell Cuckoo to use Postgres instead of SQLite. Open the $CWD/conf/cuckoo.conf file and find the [database] section. Change the connection = line to:
connection = postgresql://cuckoo:password@localhost/cuckoo

# configuring web interface 
# open reporting.conf and find the [MongoDB] section. Change enabled = no to enabled = yes.

# Also configure mode in virtualbox.conf to gui if you want to

# starting cuckoo
# run startup script

# For cuckoo rooter
cuckoo rooter --sudo --group cuckoo

# For cuckoo
cuckoo

# if for some reason a conflict occurs in package yara (installed while installing volatility) then uninstall that package and install the latest package yara-python
sudo pip uninstall yara
sudo pip install yara-python

# for web panel
cuckoo web --host 127.0.0.1 --port 8080 

commands compiled and tested by Devil
