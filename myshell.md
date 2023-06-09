# myshell

##### 安装mysql5.7

```bash
[root@master ~]# cat install_mysql5.7.sh 
#!/bin/bash
. /etc/init.d/functions #调用系统内置的函数库
MYSQL_REPO="https://dev.mysql.com/get/mysql80-community-release-el7-6.noarch.rpm"
RED="echo -e \\033[1;31m" #1类型的字体，31m红色
GREEN="echo -e \\033[1;32m" #1类型的字体，32m绿色
END="\033[0m"
MYSQL_PASSWD='shishaoxin100.'
TO_NULL='/dev/null'

check(){
	if yum list installed | grep mysql-community-server;then
		${RED}"mysql 已安装"$END
		exit
	fi
}
#下载源验证
set_repo(){
	if [ -f /etc/yum.repos.d/mysql-community.repo ];then
		${GREEN}"已有mysql源------------------"$END
		return
	else
	 ${GREEN}"配置mysql源------------------"$END
       	 yum install -y $MYSQL_REPO yum-utils > $TO_NULL
       	 yum-config-manager --disable mysql80-community > $TO_NULL
       	 yum-config-manager --enable mysql57-community > $TO_NULL
	fi
}
install(){
	set_repo
	yum module disable mysql > $TO_NULL#centos8的系统需要加，否则默认不用仓库的mysql server包
	${GREEN}"开始安装------------------"$END
	yum install -y mysql-community-server.x86_64
	if [ $? -eq 0 ];then
		${GREEN}"mysql 安装成功"$END
		rm -rf /var/lib/mysql > $TO_NULL
		systemctl start mysqld
		if systemctl is-active mysqld > $TO_NULL;then
		OLD_PASSWD=`awk '/A temporary password/{print $NF}' /var/log/mysqld.log | tail -1`
		mysqladmin -uroot -p"$OLD_PASSWD" password $MYSQL_PASSWD
		action 'mysql 初始化完成'
		else
			${RED}"mysqld启动失败，请检查服务"$END
			systemctl status mysqld
			exit
		fi
	else
		action 'mysql 安装失败' /bin/false #系统内置函数action，用于打印的时候显示特殊格式
		exit
	fi
}
check 
install
```

##### ansible-playbook生成硬件报告

```bash
tee hwreport.yml << EOF
---
- name: this play is to record
  hosts: all
  vars:
    hardware:
      - hw_name: HOST
        hw_info: "{{ ansible_hostname }}"
      - hw_name: MEMORY
        hw_info: "{{ ansible_memtotal_mb }}"
      - hw_name: BIOS
        hw_info: "{{ ansible_bios_version }}"
      - hw_name: DISK_SIZE_VDA
        hw_info: "{{ ansible_devices['vda']['size'] | default('NONE') }}"
      - hw_name: DISK_SIZE_VDB
        hw_info: "{{ ansible_devices['vdb']['size'] | default('NONE') }}"
  tasks:
    - name: this task is to record
      get_url: 
        url: http://rhgls.domainx.example.com/materials/hwreport.empty
        dest: /root/hwreport.txt
    - name: insert the info
      lineinfile:
        path: /root/hwreport.txt
        regexp: '^{{ item["hw_name"] }}='
        line: "{{ item['hw_name'] }}={{ item['hw_info'] }}"
      loop: "{{ hardware }}"
EOF
ansible-playbook hwreport.yml

```

##### 判断输入是否为数字

```bash
#!/bin/bash
is_num(){
  if[ $input =~ ^[0-9]+$ ];then
    echo 'is number'
  else
    echo 'no number'
  fi
}
```

##### 监控文件夹操作并记录在日志内

```bash
需先安装inotify-tools软件包
场景：记录目录下文件创建
#!/bin/bash
dir=/root/abc
inotifywait -mq --format %f -e create $dir |\
        while read files;do
                echo $files >> test.log
        done &

#对文件夹内文件的增删改查进行监控 
inotifywait -mrq -e 'open,modify,access,create,delete,close_write,attrib,moved_to' --timefmt '%Y-%m-%d %H:%M' --format '%T %w%f %e' /home/awk

```

##### Expect实现ssh免交互

```bash
#!/bin/bash
USER=root
PASS=123.com
IP=192.168.1.120
expect << EOF
set timeout 30
spawn ssh $USER@$IP
expect {
"(yes/no)" {send "yes\r"; exp_continue}
"password:" {send "$PASS\r"}
}
expect "$USER@*" {send "$1\r"}
expect "$USER@*" {send "exit\r"}
expect eof
EOF
```

##### 简单输出网卡信息

```bash
#!/bin/bash
interface=$1
IP=`ifconfig $interface | head -2 | tail -1 | awk '{print $2}' | awk -F":" '{print $2}'`
ZW=` ifconfig $interface | head -2 | tail -1 | awk '{print $4}' | awk -F":" '{print $2}'`
GW=`route -n | awk '$8=="'$interface'" && $4=="UG" && $3=="0.0.0.0"{print $2}'`
HN=`hostname`
DNS=`cat /etc/resolv.conf | awk '$1=="nameserver"{print $2}'`
echo '此机IP地址是' $IP
echo '此机子网掩码是' $ZW
echo '此机网关是' $GW
echo '此机主机名是' $HN
echo '此机DNS是' $DNS

```

##### PV过量防火墙自动封IP

```bash
每分钟检测一次
#!/bin/bash
log=/var/log/nginx/access.log
time=`date -d "-1 minute" +%d/%a/%Y:%H:%M`
add_iptable(){
  grep $time $log | awk '{ip[$1]++}END{for(i in ip){if(ip[i]>100){print i}}}' > ip.txt
  when read ip
  do 
    iptables -I INPUT -s $ip -j DROP 
  done < ip.txt  
}
while true
do
  add_iptable
  sleep 60
done 


11/Mar/2023:09:55:04
```

##### 部署应用到容器

```bash
#部署redis
docker pull redis
mkdir /data/redis
cd /data/redis

vi redis.conf
appendonly yes
requirepass  shixy110

#redis使用自定义配置文件启动容器（将配置文件和数据存储放到容器外部）
docker run  redis-server  /etc/redis/reids.conf  \   #自定义容器中redis的配置文件位置
-v /data/redis/redis.conf:/etc/redis/redis.conf \
-v /data/redis/data:/data  \
-p 6379:6379   \
-d --name myredis  \
redis:latest   
# java应用包含对a服务器的redis的调用
# 将java应用打包成xxx.jar可执行jar包，用java -jar xxx.jar运行，放置在target目录里
# target目录和Docakerfile在同级目录里
# 编写docker file文件
vim  Dockerfile
FROM openjdk:8-jdk-slim    #java解释器 运行环境
LABLE maintainer=shaoxin  #作者
COPY target/*.jar  /app.jar
ENTRYPOINT  ["java","-jar","/app.jar"]   #启动jar包的命令

docker build -t java-demo:v1.0 .  #构建镜像
##测试启动容器服务
docker run -d -p 8080:8080 java-demo:v1.0  #java运行的端口是8080
docker ps  #查看应用是否启动

#共享镜像
docker login
docker tag java-demo:v1.0  shaoxin/java-demo:v1.0 
docker images
docker push  shaoxin/java-demo:v1.0  #shaoxin是hub账号
#b服务器下载镜像
docker  pull  shaoxin/java-demo:v1.0  
#使用镜像运行容器
docker run -d -p 8080:8080  --name myjava java-demo:v1.0
```

##### nginx日志操作

```bash
日志格式允许包含的变量：
$remote_addr   远程地址，记录客户机 ip 地址
$remote_user    远程用户，记录客户端用户名称
[$time_local]    本地时间，服务器自身时间
$request   请求，记录请求的 URL 和 HTTP 协议版本
$status     状态，记录请求状态
$body_bytes_sent    发生给客户端的字节数\(文件大小\)，不包含响应头的大小
$http_referer    记录从哪个页面链接访问过来的\(超链接\)
$http_user_agent   记录客户端浏览器相关信息
$http_x_forwarded_for            转交 i
#分析 9 月 5 号用户访问最多的 10 个页面
grep '05/Sep/2017' cd.log | awk '{urls[$7]++} END{for(i in urls){print urls[i],i}}' | sort -k1 -rn | head -n10
#分析 9 月 5 号访问大于 100 次的 IP
grep '05/Sep/2017' cd.log | awk '{ips[$1]++} END{for(i in ips){if(ips[i]>100){print ips[i],i}}} | sort -k1 -rn'
```

##### vue编译

```bash
#!/bin/bash
set -eux
webip=36.139.159.39
#清空旧版本
rsync -a  /root/svndata/vue_test /root/svndata/project/
#拉取项目
cd /root/svndata
svn checkout svn://svn.svnbucket.com/shishaoxin100/vue_test/
#判断容器是否已创建
if docker ps -a | grep node >/dev/null
then
docker rm -f `docker ps -a | awk 'NR>1 { print $1 }'`  #删除所有容器
fi
#把旧版本依赖包拉进项目
#创建运行环境
#docker build -t  mynode .
echo "开始创建容器"
docker run --name=node -w /app -v /root/svndata/module/node_modules:/app/front/node_modules \
-v /root/svndata/vue_test/:/app:rw -id node:16.18.0

#docker run --name=node \
#  -w /app \
#-v /root/svndata/module/node_modules:/app/front/node_modules \
#  --entrypoint /bin/bash \
#  -v /root/svndata:/app/:rw \
#  -tid node:16.18.0
#编译
docker exec -it node bash -c "cd front && npm run build"
docker cp node:/app/front/dist /root/svndata/vue_test/front/
#web服务器上旧版本dist备份
ssh root@$webip 'bash -s' <<'ENDSSH'
  # 命令在web运行
   rm -rf /root/olddata/*
  mv -f /root/data/dist /root/olddata/
ENDSSH
#远程用rsync命令会报错
#ssh root@$webip  'rsync -a  /root/data/dist /root/olddata/'
#拷贝dist到目标服务器
scp -r /root/svndata/vue_test/front/dist root@$webip:/root/data/
```

##### 部署docker

```bash
yum  -y  install yum-utils
yum-config-manager  --add-repo  https://download.docker.com/linux/centos/docker-ce.repo          #添加下载仓库
yum  install -y docker-ce  docker-ce-cli  containerd.io           #安装发行版 docker、客户端命令行、容器相关组件
systemctl enable docker  --now      #开机自启 + 现在马上启动
mkdir -p  /etc/docker
#镜像仓库配置，1、阿里云镜像加速器，最好用阿里的
tee  /etc/docker/daemon.json  <<- 'EOF'
{
 "registry-mirrors": ["https://82m9ar63.mirror.aliyuncs.com"],          #此网址必须在自己的阿里云控制台-容器镜像服务找到
 "exec-opts": ["native.cgroupdriver=systemd"],
 "log-driver": "json file",
 "log-opts": { "max-size": "100m"},
 "storage-driver": "overlay2"
}
EOF
#或者2、
tee  /etc/docker/daemon.json  <<- 'EOF'
{
 "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]
}
EOF

systemctl daemon-reload
systemctl restart docker
```

##### 批量修改用户密码

```bash
# cat old_pass.txt
192.168.18.217 root 123456 22
192.168.18.218 root 123456 22
内容格式：IP User Password Port
SSH远程修改密码脚本：新密码随机生成
https://www.linuxprobe.com/books
#!/bin/bash
OLD_INFO=old_pass.txt
NEW_INFO=new_pass.txt
for IP in $(awk '/^[^#]/{print $1}' $OLD_INFO); do
USER=$(awk -v I=$IP 'I==$1{print $2}' $OLD_INFO)
PASS=$(awk -v I=$IP 'I==$1{print $3}' $OLD_INFO)
PORT=$(awk -v I=$IP 'I==$1{print $4}' $OLD_INFO)
NEW_PASS=$(mkpasswd -l 8) # 随机密码
echo "$IP $USER $NEW_PASS $PORT" >> $NEW_INFO
expect -c "
spawn ssh -p$PORT $USER@$IP
set timeout 2
expect {
\"(yes/no)\" {send \"yes\r\";exp_continue}
\"password:\" {send \"$PASS\r\";exp_continue}
\"$USER@*\" {send \"echo \'$NEW_PASS\' |passwd --stdin $USER\r
exit\r\";exp_continue}
}"
done
生成新密码文件：
# cat new_pass.txt
192.168.18.217 root n8wX3mU% 22
192.168.18.218 root c87;ZnnL 22
```

##### 屏蔽每分钟ssh登录超10次的用户

```bash
#!/bin/bash
DATE=$(date +"%b %d %H")
ABNORMAL_IP="$(tail -n10000 /var/log/auth.log |grep "$DATE" |awk
'/Failed/{a[$(NF-3)]++}END{for(i in a)if(a[i]>5)print i}')"
for IP in $ABNORMAL_IP; do
if [ $(iptables -vnL |grep -c "$IP") -eq 0 ]; then
iptables -A INPUT -s $IP -j DROP
echo "$(date +"%F %T") - iptables -A INPUT -s $IP -j DROP" >>~/ssh-loginlimit.log
fi
done
```

##### 根据日志封禁解封禁ip

```bash
#!/bin/bash
################################################################################
####
#根据web访问日志，封禁请求量异常的IP，如IP在半小时后恢复正常，则解除封禁
################################################################################
####
logfile=/data/log/access.log
#显示一分钟前的小时和分钟
d1=`date -d "-1 minute" +%H%M`
d2=`date +%M`
ipt=/sbin/iptables
ips=/tmp/ips.txt
block()
{
#将一分钟前的日志全部过滤出来并提取IP以及统计访问次数
grep '$d1:' $logfile|awk '{print $1}'|sort -n|uniq -c|sort -n > $ips
#利用for循环将次数超过100的IP依次遍历出来并予以封禁
for i in `awk '$1>100 {print $2}' $ips`
do
$ipt -I INPUT -p tcp --dport 80 -s $i -j REJECT
echo "`date +%F-%T` $i" >> /tmp/badip.log
done
}
unblock()
{
#将封禁后所产生的pkts数量小于10的IP依次遍历予以解封
for a in `$ipt -nvL INPUT --line-numbers |grep '0.0.0.0/0'|awk '$2<10 {print
$1}'|sort -nr`
do
$ipt -D INPUT $a
done
$ipt -Z
}
#当时间在00分以及30分时执行解封函数
if [ $d2 -eq "00" ] || [ $d2 -eq "30" ]
then
#要先解再封，因为刚刚封禁时产生的pkts数量很少
unblock
block
else
block
fi
```

##### 判断用户输入是否为合法ip地址

```bash
#!/bin/bash
function check_ip(){
IP=$1
if [[ $IP =~ ^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$ ]]; then
FIELD1=$(echo $IP|cut -d. -f1)
FIELD2=$(echo $IP|cut -d. -f2)
FIELD3=$(echo $IP|cut -d. -f3)
FIELD4=$(echo $IP|cut -d. -f4)
if [ $FIELD1 -le 255 -a $FIELD2 -le 255 -a $FIELD3 -le 255 -a $FIELD4 -le
255 ]; then
echo "$IP available."
else
echo "$IP not available!"
fi
else
echo "Format error!"
fi
}
check_ip 192.168.1.1
check_ip 256.1.1.1
```

##### 检测文件一致性

```bash
#!/bin/bash
######################################
检测两台服务器指定目录下的文件一致性
#####################################
#通过对比两台服务器上文件的md5值，达到检测一致性的目的
dir=/data/web
b_ip=192.168.88.10
#将指定目录下的文件全部遍历出来并作为md5sum命令的参数，进而得到所有文件的md5值，并写入到指定文
件中
find $dir -type f|xargs md5sum > /tmp/md5_a.txt
ssh $b_ip "find $dir -type f|xargs md5sum > /tmp/md5_b.txt"
scp $b_ip:/tmp/md5_b.txt /tmp
#将文件名作为遍历对象进行一一比对
for f in `awk '{print 2} /tmp/md5_a.txt'`do
#以a机器为标准，当b机器不存在遍历对象中的文件时直接输出不存在的结果
if grep -qw "$f" /tmp/md5_b.txt
then
md5_a=`grep -w "$f" /tmp/md5_a.txt|awk '{print 1}'`
md5_b=`grep -w "$f" /tmp/md5_b.txt|awk '{print 1}'`
#当文件存在时，如果md5值不一致则输出文件改变的结果
if [ $md5_a != $md5_b ]then
echo "$f changed."
fi
else
echo "$f deleted."
fi
done
```

##### 正则匹配邮箱

```bash
egrep "^[0-9a-zA-Z][0-9a-zA-Z_]{1,16}[0-9a-zA-Z]\@[0-9a-zA-Z-]*([0-9a-zA-Z])?\.
(com|com.cn|net|org|cn)$" rui

ls | egrep "^(([1-9]|[1-9][0-9]|1[0-9][0-9]|2[0-4][0-9]|25[0-5])\.){3}([0-9]|[1-9]
[0-9]|1[0-9][0-9]|2[0-4][0-9]|25[0-4])$"
```
