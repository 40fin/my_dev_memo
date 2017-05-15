Install Kafka on CentOS 7
=============

[TOC]

#### 1，前提
开始安装前，请确认服务器版本:
```bash
cd ~/install  # main directory here
uname -a
cat /etc/os-release

java -version
ls -larst `which java`
```

> 注 ： 请同时确认java是否安装配置完成，以及`JAVA_HOME`和`JRE_HOME`环境变量是否配置成功。若未完成，请先参考[这里](https://www.vultr.com/docs/how-to-install-apache-kafka-on-centos-7)完成。

我的结果是：
```bash
# uname -a
Linux iZ25225qsu2Z 3.10.0-327.3.1.el7.x86_64 #1 SMP Wed Dec 9 14:09:15 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux

# cat /etc/os-release
NAME="CentOS Linux"
VERSION="7 (Core)"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="7"
PRETTY_NAME="CentOS Linux 7 (Core)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:7"
HOME_URL="https://www.centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"

CENTOS_MANTISBT_PROJECT="CentOS-7"
CENTOS_MANTISBT_PROJECT_VERSION="7"
REDHAT_SUPPORT_PRODUCT="centos"
REDHAT_SUPPORT_PRODUCT_VERSION="7"

# java -version
openjdk version "1.8.0_65"
OpenJDK Runtime Environment (build 1.8.0_65-b17)
OpenJDK 64-Bit Server VM (build 25.65-b01, mixed mode)

# ls -larst `which java`
0 lrwxrwxrwx 1 root root 22 Dec 18  2015 /usr/bin/java -> /etc/alternatives/java
```

#### 2，步骤

##### 2.1, 下载相关文件及解压
```bash
## download
wget http://apache.org/dist/zookeeper/current/zookeeper-3.4.10.tar.gz
wget wget http://www-us.apache.org/dist/kafka/0.10.2.1/kafka_2.12-0.10.2.1.tgz

## unzip
tar -xvf zookeeper-3.4.10.tar.gz
tar -xvf kafka_2.12-0.10.2.1.tgz

## rename
sudo mv kafka_2.12-0.10.2.1 /opt
sudo mv zookeeper-3.4.10 /opt
sudo ln -s /opt/kafka_2.12-0.10.2.1 /opt/kafka
sudo ln -s /opt/zookeeper-3.4.10 /opt/zookeeper
```
##### 2.2, 访问权限配置
```bash
# for kafka
sudo useradd kafka
sudo chown -R kafka. /opt/kafka_2.12-0.10.2.1
sudo chown -h kafka. /opt/kafka

# for zookeeper
sudo useradd zookeeper
sudo chown -R zookeeper. /opt/zookeeper-3.4.10
sudo chown -h zookeeper. /opt/zookeeper
```

##### 2.3, 修改kafka配置文件
```bash
cd /opt/kafka/
sudo nano bin/kafka-server-start.sh
```

将其中的：
```bash
export KAFKA_HEAP_OPTS="-Xmx1G -Xms1G"
```
修改为：
```bash
export KAFKA_HEAP_OPTS="-Xmx512M -Xms256M"
```

##### 2.4, 配置zookeeper服务
使用如下命令，创建相应文件、目录：
```bash
# create config file from sample
sudo cp -p /opt/zookeeper/conf/zoo_sample.cfg /opt/zookeeper/conf/zoo.cfg
sudo mkdir /var/lib/zookeeper
sudo chown zookeeper. /var/lib/zookeeper

sudo nano /etc/systemd/system/zookeeper.service
```

文件`zookeeper.service`内容如下：
```bash
[Unit]
Description=Apache Zookeeper server
Documentation=http://zookeeper.apache.org
Requires=network.target remote-fs.target
After=network.target remote-fs.target

[Service]
Type=forking
User=zookeeper
Group=zookeeper
ExecStart=/opt/zookeeper/bin/zkServer.sh start
ExecStop=/opt/zookeeper/bin/zkServer.sh stop
ExecReload=/opt/zookeeper/bin/zkServer.sh restart
WorkingDirectory=/var/lib/zookeeper

[Install]
WantedBy=multi-user.target
```

##### 2.5, 配置kafka服务
```bash
sudo nano /etc/systemd/system/kafka.service
```

其内容如下：
```bash
[Unit]
Description=Apache Kafka server (broker)
Documentation=http://kafka.apache.org/documentation.html
Requires=network.target remote-fs.target
After=network.target remote-fs.target zookeeper.service

[Service]
Type=simple
User=kafka
Group=kafka
Environment=JAVA_HOME=/etc/alternatives/jre
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh

[Install]
WantedBy=multi-user.target
```

##### 2.6, 配置启动服务
```bash
sudo systemctl daemon-reload

sudo systemctl start zookeeper.service
sudo systemctl status zookeeper.service
sudo systemctl start kafka.service
sudo systemctl status kafka.service

# If you see errors then start checking your logs (/var/log/messages) and double check the service scripts.

sudo systemctl enable zookeeper.service
sudo systemctl enable kafka.service
```

##### 2.6, 配置防火墙（可选）
如果想再局域网内部其他机器访问本机的zookeeper及kafka，则需配置以下防火墙。
```bash
# check status
sudo systemctl status firewalld.service

# allow access to zookeeper and kafka ports from other computers
sudo firewall-cmd --permanent --add-port=2181/tcp
sudo firewall-cmd --permanent --add-port=9092/tcp
sudo firewall-cmd --reload
```
> 注 ： 貌似目前阿里云平台无法测试防火墙，故以上语句暂未验证。

##### 2.7, kafka PHP扩展
使用如下命令确认PHP版本，及是否已经安装了kafka
```bash
php -version
php -m|grep kafka

# install rfkafka if necessary
sudo pecl install rdkafka
```

##### 2.8, 验证
```bash
# check port of zookeeper
cat /opt/zookeeper/conf/zoo.cfg |grep clientPort

# check port of kafka
cat /opt/kafka/config/server.properties|grep listeners

# check listening port o current server
ss -ntlup

# status check
sudo systemctl status zookeeper.service
sudo systemctl status kafka.service

# create a topic
cd /opt/kafka/bin

./kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test

./kafka-topics.sh --describe --zookeeper localhost:2181 --topic test
# You should see the topic

# start a console consumer in one Shell
./kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning

# start a console producer in another Shell
./kafka-console-producer.sh --broker-list localhost:9092 --topic test
```

#### 3，参考资料

- [How to Install Apache Kafka on CentOS 7](https://www.vultr.com/docs/how-to-install-apache-kafka-on-centos-7)
- [Installing Apache Kafka and Zookeeper CentOS 7.2](http://davidssysadminnotes.blogspot.com/2016/01/installing-apache-kafka-and-zookeeper.html)
- [Installing the extension with PECL](https://arnaud-lb.github.io/php-rdkafka/phpdoc/rdkafka.installation.pecl.html)
