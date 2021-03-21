### LAMP stack installation for WordPress

### Resources
  - Digital Ocean account
  - Webserver Server machine (CentOS 7.6)
  - Database machine (Ubuntu 18.04)

### Description

Installing, updating and removing packages on every distributon of Linux can come with it's own challenges , especially when you just got into IT world. Every Linux distribution has it's own package manager for example:

- Debian ```dpkg```(Debian package tool).
- Ubuntu (based on the Debian Linux distribution) uses ```apt```(Advanced Package Tool). 
- Red Hat (Red Hat-based Linux systems) like CentOS uses ```yum``` (yellow dog updater, modifier). 
- Fedora also (Red Hat based Linux distro) uses ```yum```and ```dnf```(Dandified yum next generation of yum).

Other Operating systems like Mac OS and Windows also have their own package manager:

- Mac OS (based on the Unix operating system) uses ```homebrew```. 
- Windows OS uses ```winget```.

The commands dpkg and rpm operate on individual package files. In practice, almost all package management tasks are performed by the commands ```apt``` on systems that use DEB packages or by ```yum``` or ```dnf``` on systems that use RPM packages. These commands work with catalogues of packages, can download new packages and their dependencies, and check for newer versions of the installed packages. More often than not, a package may depend on others to work as intended.
However, installing packages depends on what interface you are comfortable working with either GUI, CLI or simply use a tool like an ```Ansible```, and get it done with one playbook. The advantages of ansible playbooks is that, once you write a playbook it is reusable, and you can adjust it, depending what package you want to install, just pass it on variables file, without touching an actual playbook. 
In our case we will use ```Ansible``` and we are working with CentOS 7.6 as our ```webserver``` and Ubuntu 18.04 as our ```database```, while intalling a LAMP (Linux, Apache, MySql and PHP) stack, we faced some issues, we LAMP for WordPress, because we want to create our website using WordPress. 
Before we do anything we check ```Who we are?``` and ```Where we are?```that is ```golden rule``` of using configuration management tool ```Ansible```.  We use ansible modules for writing a playbooks but, before we do automation we have know how to do it on command line, then it will be much easier for us to build our playbook's tasks in order make it ```sequential```.  Ansible runs tasks in order, that is why we need to know, which task needs to be run first and when we do it on CLI, it will be more clear for us, all commands which we run on CLI is here:
```
sudo yum install epel-release
sudo yum install http://rpms.remirepo.net/enterprise/remi-release-7.rpm
sudo yum install yum-utils
sudo yum-config-manager --enable remi-php72
sudo yum update
sudo yum install php72
sudo yum install php72-php-fpm php72-php-gd php72-php-json php72-php-mbstring php72-php-mysqlnd php72-php-xml php72-php-xmlrpc php72-php-opcache
```
To start with we installed wget, curl, and vim while our machines were booting and we listed package names for ```webserver``` and ```database``` in variables file: 
#### webserver packages
- epel-release
- mysql
- httpd 

#### database packages:
- mariadb-server

and we used ```yum``` module for it. All went well no issues! Until we installed ```php``` package, where installed version of php is 5.4 and it's a latest version that ```yum repository``` had.  But we know, this version of ```php``` isn't compitable with WordPress it's too old, it has to be at least php 5.6 and up. So to fix that we need to install third party repositories, where the latest version of php is 7.2:

- epel-release (Extra Packages for Enterprise Linux)
- remi-php72 ( a large collection of RPMS, including latest versions of PHP, Remi Collet is author at Red Hat Developer) 

Why it's two different repositories? Because we also want to install all dependencies of ```php``` and they are in both repositories. After installing listed above repos we are able to install the php 7.2 version with dependencies.

Since we don't need  php 5.4 we remove it with command ```yum remove php* -y```. Keep in mind, when you install packages or repository they all come with dependencies and they do get installed in different places on your machine and the main configuration file for ```yum``` is under /etc/yum.conf, and all the repos are under /etc/yum.repos.d. It's just the repository file names not the actual repository itself. When you open it you can see either metalink, mirrorlist or baseurl to those repositories, using those links you are able to istall needed packages. If you decide for some reason to remove repository you always give ```*``` after the name of package or repo you want to delete. The next image is how it looks yum.repos.d. from inside:

<img src="images/yum.repo.d_content.png" alt="aws" width="800" height="450">

For a security reasons always update your packages to latest versions, in my case I checked all  installed packages versions, before to download the WordPress it is important, you need to make sure that  all packages are compatible with each other.
```
ansible -i inventory.yaml webserver -m command -a "mysql --version"
ansible -i inventory.yaml webserver -m command -a "httpd -v"
ansible -i inventory.yaml webserver -m command -a "php72 --version"
ansible -i inventory.yaml database -m command -a "mariadb --version"
ansible -i inventory.yaml database -m command -a "mysql --version"
ansible -i inventory.yaml webserver -m command -a "mysql --version"
```
Output from that commands: 
```
Server version: Apache/2.4.6 (CentOS)
PHP 7.2.34 (cli)

mariadb  Ver 15.1 
mysql client:

database | CHANGED | rc=0 >>
mysql  Ver 15.1 Distrib 10.1.47-MariaDB, for debian-linux-gnu (x86_64) using readline 5.2

webserver | CHANGED | rc=0 >>
mysql  Ver 15.1 Distrib 5.5.68-MariaDB, for Linux (x86_64) using readline 5.1
```
After installing WordPress we faced another issue, ```wordpress``` wasn't reading from /var/www/html/index.html, to fix that we tried to install extra packages for PHP still didn't work. The next step was to check the version of ```wordpress``` with command:
```
grep wp_version /var/www/html/wp-includes/version.php
```
You have to run it from the inside of ```webserver``` and we found out that the version of installed ```wordpress``` is 5.7. With that info in our hands, we did some more research on WordPress, and found out that WordPress 5.7 is not compitable with PHP 7.2. That was an eye opening for most of us, another rule comes around ```Always check compitablity of packages!``` With that in mind we sshed as a root user to ```webserver``` and deleted all ```php``` packages, next we disabled remi-php72 repo, after we enabled remi-php7.3 and installed PHP 7.3 with all it's dependencies, after that long journey our wordpress was reading from index.html. 
But again before we adjusted our playbook tasks we run all the commands on command line:
```
yum-config-manager --disable remi-php72
yum-config-manager --enable remi-php73
yum install php php-common php-opcache php-mcrypt php-cli php-gd php-curl php-mysql –y
```

We installed mariadb on database droplet, but we have to still run a secure installation, in order to do that we run ```mysql_secure_installation``` it's run by root user from your local machine after ssh into it. It will ask you to enter root user’s password, but since we didn't created root user's password yet, just press enter and set root's password for database.  you need to run:
It will ask you to enter root user’s password for the server,  but we database itself. If you have installed and haven’t set the root password yet , then just press enter,  set root password yes. Now it's time to configure database on the browser when we put an IP address of the webserver, we have to use information from database configuration when we creatd a database. To do that we run the next command:  
```
mysql -u root -p
```
Enter roots password and create a database, database user from webserver with private IP address of webserver. To ```database``` we can access from ```webserver``` machine with the next command, where -u is a user and -p is a password of the user, in this case it's a root user.After getting inside of ```database``` machine we created ```wordpress database``` , ```admin user with password``` and granted all priveleges to our admin user (save that info somewhere we are going to need it when we configure WordPress). 

### Useful Links

[Fedora distribution package manager](https://fedoraproject.org/wiki/DNF?rd=RPM)

[Linux package management with YUM and RPM](https://www.redhat.com/sysadmin/how-manage-packages)

[LAMP stack installation on CentOS 7 with liquidweb](https://www.liquidweb.com/kb/install-lamp-stack-centos-7/)

[LAMP stack installation on CentOS 7 with linuxize](https://linuxize.com/series/install-lamp-stack-on-centos-7/)

[Remi repository](http://rpms.remirepo.net/)

[Epel-release repository](https://fedoraproject.org/wiki/EPEL)

[PHP Installation](https://www.scriptcase.net/docs/en_us/v9/manual/02-scriptcase-installation/06-linux_php/)

### Notes

When you want to install a package from repository like ```yum``` make sure to update it first, that way your packages will be up to date.
Remember that WordPress 5.7 is not compitable with PHP 7.2, that's why it didn't work.
