# openssh openssl 升级

> Telnet协议是TCP/IP协议族中的一员，是Internet远程登录服务的标准协议和主要方式。它为用户提供了在本地计算机上完成远程主机工作的能力。升级过程防止升级失败，需要先开启telnet服务，防止升级失败连接不上远程主机

## telnet服务

1、安装依赖

```
yum -y install xinetd  telnet-server
```

2、启动服务

```
systemctl start telnet.socket 
systemctl start xinetd

ss -anpt | grep 23 检查23端口是否开启
```

3 配置tty

```
cp /etc/securetty{,.bak}
vim /etc/securetty
# 最后添加，为了能用root登录
pts/0
pts/1
```

升级之前必须用确保telnet 能连接成功，升级成功后 

关闭服务

```
systemctl stop telnet.socket 
systemctl stop xinetd


mv /etc/securetty.bak /etc/securetty
```



## 编译安装

Zlib 最新版地址 

https://nchc.dl.sourceforge.net/project/libpng/zlib/1.2.11/zlib-1.2.11.tar.gz

Openssl 地址  wget https://www.openssl.org/source/openssl-1.1.1g.tar.gz

Openssh地址

https://cdn.openbsd.org/pub/OpenBSD/OpenSSH/portable/openssh-8.8p1.tar.gz

 

***\*先安装openssl\**** ***\*， openssh\**** ***\*， zlib\****



```shell
Ubuntu 依赖安装 apt-get install gcc make
```

 

1 、Openssl  安装

```shell
centos：
yum -y install openssl-devel pcre-devel  pam-devel  gcc-c++ zlib-devel perl

tar  -zxvf openssl-1.1.1g.tar.gz

cd openssl-1.1.1g
 
./config --prefix=/usr/local/openssl

make && make install

 

ldd /usr/local/openssl/bin/openssl   #检查函数库


echo "/usr/local/openssl/lib" >> /etc/ld.so.conf  #添加所缺函数库


ldconfig -v    #更新函数库

openssl/bin/openssl version   #查看新安装的版本


which openssl      #查看旧版本openssl命令在哪里

mv /bin/openssl /usr/bin/openssl.old   #将旧版本openssl移除


Ubuntu： mv /usr/bin/openssl /usr/bin/openssl.bak

ln -sf /usr/local/openssl/bin/openssl /usr/bin/openssl     #新版本制作软链接

cd /usr/bin/

cp /usr/local/openssl/bin/openssl ./

openssl version     最后查看版本，更新完毕
```

2 、openssh  安装

 

```shell
tar -zxvf openssh-8.8p1.tar.gz 
cd openssh-8.8p1
./configure --prefix=/usr --sysconfdir=/etc/ssh --with-zlib --without-openssl-header-check --with-ssl-dir=/usr/local/openssl/  --with-privsep-path=/var/lib/sshd

Make

rpm -e --nodeps `rpm -qa | grep openssh`

make install

cp -p contrib/redhat/sshd.init /etc/init.d/sshd

chmod +x /etc/init.d/sshd

vim /etc/ssh/sshd_config	PermitRootLogin yes

​                PasswordAuthentication yes

​                PubkeyAuthentication yes

 
setenforce 0

Sshd -t

Chomd 600 /etc/ssh/ssh_host_ecdsa_key 等

ssh -V


Ubuntu  安装：
apt-get install libpam0g-dev

mkdir /tmp/ssh_bak/init.d -p

cp -r /etc/ssh /tmp/ssh_bak

cp /etc/init.d/ssh /tmp/ssh_bak/init.d

service sshd stop

dpkg --purge openssh-server  #卸载

apt-get install build-essential libtool automake Zlib*

cd ../openssh-8.8p1

./opt/openssh-8.8p1/configure --prefix=/usr/local/ssh --sysconfdir=/etc/ssh --with-ssl-dir=/usr/local/openssl/ --without-openssl-header-check

 make

make install

echo "sshd:x:74:74::/var/run/sshd:/sbin/nologin" >>/etc/passwd

echo "export LANG=zh_CN.UTF-8" >>/etc/profile

cp /usr/local/ssh/bin/* /usr/bin/

cp /usr/local/ssh/sbin/sshd /usr/bin/sshd

mv /etc/init.d/ssh.old /etc/init.d/ssh

ln -s /usr/local/ssh/sbin/sshd /usr/sbin/sshd

systemctl enable ssh

systemctl restart ssh

ssh -V

 

报错：Failed to start ssh.service: Unit ssh.service is masked.

 解决方法：systemctl unmask ssh

Removed /etc/systemd/system/ssh.service.

 #原因可能是之前使用apt-get 安装过ssh，服务被标记过
```



3、zlib  安装

```shell
yum install  -y gcc make

mv zlib-1.2.11.tar.gz  /opt/

cd /opt/

tar -zxvf zlib-1.2.11.tar.gz 

cd zlib-1.2.11

./configure --libdir=/lib64/ 

make 

make install

ll /lib64/libz*

rpm -e zlib-1.2.7-19.el7_9.x86_64 --nodeps

ldconfig -v
```



## rpm 安装

从 github 已zip的方式下载 https://github.com/boypt/openssh-rpms，上传到服务器
>如果openssh 7.X 版本升级前把 /etc/ssh/sshd_config 备份

指定版本,修改version.env即可

```shell
cat version.env
OPENSSLSRC=openssl-3.0.12.tar.gz #指定openssl包版本
OPENSSHSRC=openssh-9.5p1.tar.gz  #指定openssh包版本
ASKPASSSRC=x11-ssh-askpass-1.2.4.1.tar.gz
PKGREL=1

OPENSSHVER=${OPENSSHSRC%%.tar.gz}
OPENSSHVER=${OPENSSHVER##openssh-}
OPENSSLVER=${OPENSSLSRC%%.tar.gz}
OPENSSLVER=${OPENSSLVER##openssl-}

```

构建准备

```shell
yum groupinstall -y "Development Tools"
yum install -y imake rpm-build pam-devel krb5-devel zlib-devel libXt-devel libX11-devel gtk2-devel perl-IPC-Cmd wget
```

开始打rpm包

```shell
./pullsrc.sh #将会下载源码包 到 当前的downloads目录下

#报错，可能会遇到
要以不安全的方式连接至 www.openssl.org，使用“--no-check-certificate”。
Aborted, error 5 in command: wget $OPENSSLMIR/$OPENSSLSRC

#解决
修改 pullsrc.sh 将几个wget 添加 --no-check-certificate 选项
sed -i "s/wget/wget --no-check-certificate/g" pullsrc.sh #更改后重新执行即可


./compile.sh #允许脚本构建 rpm包，执行后会rpm包的路径，
cd el7/RPMS/x86_64/ 进入目录也可以把这里面的rpm 传输到其他离线机器上
rpm -Uivh ./*.rpm #进行升级
systemctl status sshd #确保sshd 起来了，如果是 openssh 7.x 升级的，要把原先备份的 /etc/ssh/sshd_config 恢复

#报错，可能会遇到
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
12月 28 18:41:42 localhost.localdomain sshd[88555]: Permissions 0640 for '/etc/ssh/ssh_host_ed25519_key' are too open.
12月 28 18:41:42 localhost.localdomain sshd[88555]: It is required that your private key files are NOT accessible by others.
12月 28 18:41:42 localhost.localdomain sshd[88555]: This private key will be ignored.
12月 28 18:41:42 localhost.localdomain sshd[88555]: sshd: no hostkeys available -- exiting.

解决
chmod 600 /etc/ssh/ssh_host_ed25519_key
```

