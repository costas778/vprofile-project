# ![App Screenshot](https://raw.githubusercontent.com/costas778/vprofile-project/main/vp1.png)

# **Project-1: Vprofile Project: Multi Tier Web Application Stack Setup Locally**

---

## **Prerequisite**

**VPROFILE PROJECT SETUP**

- Oracle VM Virtualbox  
- Vagrant  
- Vagrant plugins:

```bash
vagrant plugin install vagrant-hostmanager
vagrant plugin install vagrant-vbguest
```

- Git bash and a CLI (I used PowerShell Administrator)

---

## **Step1: VM Setup**

Clone the repository:

```bash
git clone https://github.com/costas778/vprofile-project.git
```

Install the vagrant host manager plugin:

```bash
vagrant plugin install vagrant-hostmanager
```

Power up your VMs:

```bash
vagrant up
```

> NOTE: Bringing VMs can take long time sometimes. If VM setup stops in the middle, run `vagrant up` again.

Check your VMs in Oracle VM VirtualBox Manager.

### Validate VMs:

```bash
vagrant ssh web01
cat /etc/hosts
ping app01
logout

vagrant ssh app01
cat /etc/hosts
ping rmq01
ping db01
ping mc01
logout
```

![App Screenshot](https://raw.githubusercontent.com/costas778/vprofile-project/main/vp2.png)

![App Screenshot](https://raw.githubusercontent.com/costas778/vprofile-project/main/vp3.png)

---

## **Step2: Provisioning**

### **Services**

1. Nginx: Web Service  
2. Tomcat: Application Server  
3. RabbitMQ: Broker/Queuing Agent  
4. Memcache: DB Caching  
5. ElasticSearch: Indexing/Search service  
6. MySQL: SQL Database

![App Screenshot](https://raw.githubusercontent.com/costas778/vprofile-project/main/vp4.png)

---

## **Provisioning MySQL (db01)**

```bash
vagrant ssh db01
sudo su -
yum update -y
DATABASE_PASS='admin123'
vi /etc/profile
source /etc/profile
yum install epel-release -y
yum install git mariadb-server -y
systemctl start mariadb
systemctl enable mariadb
systemctl status mariadb
mysql_secure_installation
```

Check connectivity:

```bash
mysql -u root -p
exit
```

![App Screenshot](https://raw.githubusercontent.com/costas778/vprofile-project/main/vp5.png)

Clone repo and initialize DB:

```bash
git clone https://github.com/costas778/vprofile-project.git
cd vprofile-project/src/main/resources/

mysql -u root -p"$DATABASE_PASS" -e "create database accounts"
mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'app01' identified by 'admin123'"
cd ../../..
mysql -u root -p"$DATABASE_PASS" accounts < src/main/resources/db_backup.sql
mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"
```

Verify DB:

```bash
mysql -u root -p"$DATABASE_PASS"
show databases;
use accounts;
show tables;
exit
```

Restart:

```bash
systemctl restart mariadb
logout
```

---

## **Provisioning Memcache (mc01)**

```bash
vagrant ssh mc01
sudo su -
yum update -y
yum install epel-release -y
yum install memcached -y
systemctl start memcached
systemctl enable memcached
memcached -p 11211 -U 11111 -u memcached -d
ss -tunlp | grep 11211
exit
```

---

## **Provisioning RabbitMQ (rmq01)**

```bash
vagrant ssh rmq01
sudo su -
yum update -y
yum install epel-release -y
yum install wget -y
cd /tmp/
wget http://packages.erlang-solutions.com/erlang-solutions-2.0-1.noarch.rpm
rpm -Uvh erlang-solutions-2.0-1.noarch.rpm
curl -s https://packagecloud.io/install/repositories/rabbitmq/rabbitmq-server/script.rpm.sh | bash
yum install rabbitmq-server -y
systemctl start rabbitmq-server
systemctl enable rabbitmq-server

cd ~
echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config
rabbitmqctl add_user test test
rabbitmqctl set_user_tags test administrator
systemctl restart rabbitmq-server
systemctl status rabbitmq-server
exit
```

---

## **Provisioning Tomcat (app01)**

```bash
vagrant ssh app01
sudo su -
yum update -y
yum install epel-release -y
yum install java-1.8.0-openjdk git maven wget -y
cd /tmp
wget https://archive.apache.org/dist/tomcat/tomcat-8/v8.5.37/bin/apache-tomcat-8.5.37.tar.gz
tar xzvf apache-tomcat-8.5.37.tar.gz
useradd --home-dir /usr/local/tomcat8 --shell /sbin/nologin tomcat
cp -r /tmp/apache-tomcat-8.5.37/* /usr/local/tomcat8/
chown -R tomcat:tomcat /usr/local/tomcat8
```

Create tomcat service at `/etc/systemd/system/tomcat.service`:

```ini
[Unit]
Description=Tomcat
After=network.target

[Service]
User=tomcat
WorkingDirectory=/usr/local/tomcat8
Environment=JRE_HOME=/usr/lib/jvm/jre
Environment=JAVA_HOME=/usr/lib/jvm/jre
Environment=CATALINA_HOME=/usr/local/tomcat8
Environment=CATALINE_BASE=/usr/local/tomcat8
ExecStart=/usr/local/tomcat8/bin/catalina.sh run
ExecStop=/usr/local/tomcat8/bin/shutdown.sh
SyslogIdentifier=tomcat-%i

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable tomcat
systemctl start tomcat
systemctl status tomcat
```

Build and deploy WAR:

```bash
cd /tmp
git clone https://github.com/costas778/vprofile-project.git
cd vprofile-project/
vi src/main/resources/application.properties
```

Set:

```properties
jdbc.url=jdbc:mysql://db01:3306/accounts
jdbc.username=admin
jdbc.password=admin123
memcached.active.host=mc01
memcached.active.port=11211
memcached.standBy.host=127.0.0.2
memcached.standBy.port=11211
rabbitmq.address=rmq01
rabbitmq.port=5672
rabbitmq.username=test
rabbitmq.password=test
```

```bash
mvn install
cd target/
systemctl stop tomcat
rm -rf /usr/local/tomcat8/webapps/ROOT
cp vprofile-v2.war /usr/local/tomcat8/webapps/ROOT.war
systemctl start tomcat
```

---

## **Provisioning NGINX (web01)**

```bash
vagrant ssh web01
sudo apt update && sudo apt upgrade -y
sudo apt install nginx -y
sudo su -
vi /etc/nginx/sites-available/vproapp
```

```nginx
upstream vproapp {
  server app01:8080;
}

server {
  listen 80;
  location / {
    proxy_pass http://vproapp;
  }
}
```

```bash
rm -rf /etc/nginx/sites-enabled/default
ln -s /etc/nginx/sites-available/vproapp /etc/nginx/sites-enabled/vproapp
systemctl restart nginx
```

---

## **Validate Application**

```bash
ifconfig
```

Open browser: `http://<web01_ip>`

- Validate DB with `admin_vp` / `admin_vp`
- Validate app from Tomcat
- Validate RabbitMQ and Memcache
- Confirm data is cached on second load

---

## **CleanUp**

```bash
vagrant destroy
