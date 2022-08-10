# Before starting make sure virtualization is enabled on the host
# And make sure if your installing ubuntu as the main OS then must disable secure boot

Add a new user 
    
    sudo adduser cuckoo
Set password and other configurations

Add new user to group sudo
    
    sudo usermod -aG sudo cuckoo

Install Initial Requirements
    
    sudo apt-get -y install python python3-pip python-dev virtualenv python-setuptools curl git libffi-dev libssl-dev libjpeg-dev zlib1g-dev swig

Install mongodb For web Interface

    sudo apt-get -y install mongodb

Install postgresql For database

    sudo apt-get -y install postgresql postgresql-contrib libpq-dev

Install pydeep (optional but recommended)

    sudo apt-get -y install libfuzzy-dev ssdeep
    sudo ldconfig
    git clone https://github.com/kbandla/pydeep.git
    cd pydeep
    python setup.py build
    python setup.py test
    sudo python setup.py install
    cd ..
    rm -r pydeep

Install python2 and pip2

    sudo apt-get -y install python2
    curl https://bootstrap.pypa.io/pip/2.7/get-pip.py --output get-pip.py
    sudo python2 get-pip.py

Install valatility (optional but recommended)
    
    sudo python2 -m pip install -U distorm3 yara pycrypto pillow openpyxl ujson pytz ipython capstone
    sudo ln -s /usr/local/lib/python2.7/dist-packages/usr/lib/libyara.so /usr/lib/libyara.so
    sudo python2 -m pip install -U git+https://github.com/volatilityfoundation/volatility.git

Install Virtualbox
    
    sudo apt-get -y install virtualbox

Add user cuckoo to group vboxusers
    
    sudo usermod -a -G vboxusers cuckoo

Install tcpdump

    sudo apt-get -y install tcpdump apparmor-utils
    sudo aa-disable /usr/sbin/tcpdump

Add user cuckoo to group pcap
    
    sudo groupadd pcap
    sudo usermod -a -G pcap cuckoo
    sudo chgrp pcap /usr/sbin/tcpdump
    sudo setcap cap_net_raw,cap_net_admin=eip /usr/sbin/tcpdump

Verfication of last Command
    
    getcap /usr/sbin/tcpdump
/usr/sbin/tcpdump = cap_net_admin,cap_net_raw+eip

logout and login as new user

Create a virtual environment and install cuckoo and vmcloak
    
    virtualenv -p python2.7 env
    source env/bin/activate
    sudo pip2 install cuckoo vmcloak

Start cuckoo to setup default configuration and download signatures of known files

    cuckoo init
    cuckoo community

# Setting up guest VMs for cuckoo
Download windows 7 iso file

    wget https://cuckoo.sh/win7ultimate.iso
In case of certificate error

    wget https://cuckoo.sh/win7ultimate.iso --no-check-certificate

Mount iso file to the directory **/mnt/win7** (default directory where vmcloak looks for setups)

    sudo mkdir /mnt/win7
    sudo mount -o ro,loop win7ultimate.iso /mnt/win7

Create network for virtual machines (in virtual environment)
    
    vmcloak-vboxnet0

Network Configuration
Note: Replace eth0 with network interface adaptor of your host that is currently active

    sudo sysctl -w net.ipv4.conf.vboxnet0.forwarding=1
    sudo sysctl -w net.ipv4.conf.eth0.forwarding=1

Set iproutes for vboxnet0 (global routing)

    sudo iptables -t nat -A POSTROUTING -o eth0 -s 192.168.56.0/24 -j MASQUERADE
    sudo iptables -P FORWARD DROP
    sudo iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
    sudo iptables -A FORWARD -s 192.168.56.0/24 -j ACCEPT
    sudo iptables -A FORWARD -s 192.168.56.0/24 -d 192.168.56.0/24 -j ACCEPT
    sudo iptables -A FORWARD -j LOG

Write a script for above commands (Network Configuration) and run that script everytime before starting cuckoo

Install windows 7

    vmcloak init --verbose --win7x64 win7x64base --cpus 2 --ramsize 2048

Clone windows 7 (so that we can create more copies in future using base version)

    vmcloak clone win7x64base win7x64cuckoo

VMCloak supports the installation of multiple software packages. A full list of supported packages and versions can be listed:

    vmcloak list deps

A software package can be installed with the following syntax

    vmcloak install <image name> <package>
    
A specific version can be provided by adding

    package.version=X
If no version is selected, the default version will be picked. However, it is better to specify versions with packages as most of the default packages are outdated.

Serialkey can be provided by adding

    package.serialkey=X

We will be installing some basic software packages:
    
    vmcloak install win7x64cuckoo pillow python27 adobepdf adobepdf.version=11.0.19

Optional step: Installing a Microsoft Office version so that Office document can be analyzed. Office 2007 is most likely to work, some builds of higher versions of Office sometimes cause issues with the Cuckoo MonitorCuckoo Monitor:

    vmcloak install win7x64cuckoo office office.version=2007 office.isopath=/path/to/office2007.iso office.serialkey=XXXXX-XXXXX-XXXXX-XXXXX-XXXXX

VMCloak will register a VirtualBox VM for each snapshot created. After snapshotting, it is no longer possible to change the image. The syntax of the snapshot command is

    vmcloak snapshot <options> <image name> <vmname> <ip to use>

Using the --count parameter, we can create multiple snapshots at once. Let’s create four
    
    vmcloak snapshot --count 4 win7x64cuckoo 192.168.56.101

If encounter multi-attach disk vm error then fix it manually (covered in the detailed report attached)

The above command will create VMs win7x64cuckoo1-4 with IPs 192.168.56.101-104.
After VMCloak is finished, the VMs can be listed using:
    
    vmcloak list vms

Add guest VMs to cuckoo (this command while automatically add guest VMs to virtualbox.conf)

    while read -r vm ip; do cuckoo machine --add $vm $ip; done < <(vmcloak list vms)

NOTE: By default, Cuckoo Current Working Directory (CWD) is **/home/user/.cuckoo** where as all the configuration files are located in **conf** directory inside CWD.

After adding guest VMs remove cuckoo1 from virtualbox.conf (exists there by default)
Remove all the content from line **[cuckoo1]** to line **osprofile =** and cuckoo1 from list of vms

Configure Per Network Analysis
In order to enable direct internet to analysis VM, open the /etc/iproute2/rt_tables file and roll a random number that is not yet present in this file with your dice of choice and use it to craft a new line at the end of the file e.g; "400    eth0" 

For Internet Routing
open routing.conf and write eth0 in front of internet

Setup Postgres as DBMS

    sudo pip install psycopg2

    sudo -u postgres psql
    CREATE DATABASE cuckoo;
    CREATE USER cuckoo WITH ENCRYPTED PASSWORD 'password';
    GRANT ALL PRIVILEGES ON DATABASE cuckoo TO cuckoo;
    \q

Open the $CWD/conf/cuckoo.conf file and find the **[database]** section. Change the **connection =** line to

    connection = postgresql://cuckoo:password@localhost/cuckoo

Configure web interface 
open reporting.conf and find the **[MongoDB]** section. Change **enabled = no** to **enabled = yes**.

Also configure mode in virtualbox.conf to gui if you want to

Starting cuckoo
Run script to set iproutes and enable traffic forwarding

Start Cuckoo Rooter

    cuckoo rooter --sudo --group cuckoo

Start Cuckoo

    cuckoo

If for some reason a conflict occurs in package yara (installed while installing volatility) then uninstall that package and install the latest package yara-python

    sudo pip uninstall yara
    sudo pip install yara-python

Start web panel

    cuckoo web --host 127.0.0.1 --port 8080 

# **commands compiled and tested by Devil**
