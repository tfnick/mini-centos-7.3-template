# 项目简单说明
记录centos7.3镜像制作过程，以及后期运维的操作方法

# 虚拟化软件安装
调研了kvm，vsphere之后，选择vsphere作为虚拟化平台软件

# 操作系统安装
选择centos7.3-1611版本，主要是好雨平台使用此版本，此版本作为以后私有化以及公有化部署的参考。

[下载地址](https://pan.baidu.com/s/1nwjp5FF)

+ 1核心CPU
+ 2G内存
+ 20G硬盘（以上三者后期均可扩容）
+ 选择最小化安装
+ 确认关闭selinux（默认firewalld开启，iptalbes没有安装）
```jshelllanguage
/usr/sbin/sestatus -v 

SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      28
```
```jshelllanguage
vi /etc/selinux/config
```
> 将SELINUX=enforcing改为SELINUX=disabled
```jshelllanguage
reboot
```

+ 配置静态IP
```shell
vi /etc/sysconfig/network-scripts/ifcfg-ens160

TYPE=Ethernet
BOOTPROTO=static
IPADDR=192.168.1.170
PREFIXX=24
GATEWAY=192.168.1.1
DNS1=8.8.8.8
NAME=ens160
DEVICE=ens160
ONBOOT=yes

```
> 有时候安装完之后的网络配置文件为ifcfg-ens32

+ 时区同步
```jshelllanguage
sudo yum install ntp
timedatectl set-timezone Asia/Shanghai
timedatectl set-ntp yes
timedatectl
```


# 后期运维

## 免密码登陆脚本
+ auto_ssh.ssh
```jshelllanguage
#!/usr/bin/expect  
set timeout 10  
set username [lindex $argv 0]  
set password [lindex $argv 1]  
set hostname [lindex $argv 2]  

spawn ssh-copy-id -i /root/.ssh/id_rsa.pub $username@$hostname

expect {
            #first connect, no public key in ~/.ssh/known_hosts
            "Are you sure you want to continue connecting (yes/no)?" {
            send "yes\r"
            expect "password:"
                send "$password\r"
            } 
            #already has public key in ~/.ssh/known_hosts
            "password:" {  
                send "$password\r"
            } 
            "Now try logging into the machine" {
                #it has authorized, do nothing!
            }
       }
expect eof
```
+ 使用脚本
```jshelllanguage
chmod +x auto_ssh.ssh
./auto_ssh.ssh root PASSWORD 192.168.1.100
```

## 开放内网http下载服务
+ 配置域名A记录映射到硬件防火墙的公网地址
+ 登陆硬件防火墙，配置虚拟服务器以及端口触发规则，比如开放8001端口
+ nginx服务器上的防火墙firewalld也需要开放8001端口
```jshelllanguage
#查看状态
systemctl status firewalld
#列出已经开放端口
firewall-cmd --list-ports
#增加开放端口
firewall-cmd --zone=public --add-port=8001/tcp --permanent
firewall-cmd --zone=public --add-port=80/tcp --permanent
firewall-cmd --reload
#停止
systemctl stop firewalld
```
+ nginx1.10.1下载服务器配置
> wget -c http://nginx.org/download/nginx-1.10.1.tar.gz

> tar zxvf nginx-1.10.1.tar.gz

> cd nginx-1.10.1 && ./configure && make && make install

> vi /usr/local/nginx/conf/nginx.conf

```nginx
    server {
       listen 8001;
       charset utf-8;
       server_name localhost;
       #root /usr/local/nginx/html;
       location / { 
	  root /usr/local/nginx/html;   
          if ($request_filename ~* ^.*?\.(txt|xml|doc|pdf|rar|gz|zip|docx|exe|xlsx|ppt|pptx)$){
              add_header Content-Disposition: 'attachment;';
          }
       }
       autoindex on;
       autoindex_exact_size off;
       autoindex_localtime on;
    }

```

+ 开机启动

> systemctl enable nginx.service

> systemctl enable firewalld.service

## FAQ
+ rpm依赖下载
> yum install -y  ./rpm/*.rpm --downloadonly --downloaddir=./rpm/

+ 内核版本
> cat /etc/redhat-release


## TODO
+ nginx 配置ssl 下载
+ 研究docker版本 Docker version 17.12.0-ce, build c97c6d6
+ 制作应用镜像
