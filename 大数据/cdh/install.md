# 安装cdh-6.3.1

## 安装cdh之前需要做的准备

1. 评估存储空间 https://docs.cloudera.com/documentation/enterprise/6/latest/topics/cm_ig_reqs_space.html#concept_tjd_4yc_gr
2. 配置网络名 https://docs.cloudera.com/documentation/enterprise/6/latest/topics/configure_network_names.html#configure_network_names

### 配置网络名

> **Important:**
>
> - The canonical name of each host in /etc/hosts **must** be the FQDN (for example myhost-1.example.com), not the unqualified hostname (for example myhost-1). The canonical name is the first entry after the IP address.
> - Do not use aliases, either in /etc/hosts or in configuring DNS.
> - Unqualified hostnames (short names) must be unique in a Cloudera Manager instance. For example, you cannot have both host01.example.com and host01.standby.example.com managed by the same Cloudera Manager Server.

1. 配置hostname为唯一的名（不允许为localhost）

```
sudo hostnamectl set-hostname cdh01.cdhserver.com
```

2. 修改`/etc/hosts`

```
192.168.43.129 cdh01.cdhserver.com cdh01
192.168.43.131 chd02.cdhserver.com cdh02
192.168.43.132 chd03.cdhserver.com cdh03
```

3. Edit` /etc/sysconfig/network` with the FQDN of this host only

```
HOSTNAME=cdh01.cdhserver.com
```

4. 验证

a. 执行`uname -a` 出现配置的`hostname`

b. 执行`ifconfig` hosts 网卡`eth0`或者其他匹配的ip

c. 执行`host -v -t A $(hostname)` 匹配到`hostname`以及`ifconfig`中的`eth0`或者其他

```
[root@cdh01 ~]# host -v -t A cdh01.cdhserver.com
Trying "cdh01.cdhserver.com"
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 23298
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;cdh01.cdhserver.com.		IN	A

;; ANSWER SECTION:
cdh01.cdhserver.com.	5	IN	A	198.18.1.154

Received 72 bytes from 192.168.43.2#53 in 1 ms

```

### 禁用防火墙

1. 备份iptables规则

```
sudo iptables-save > ~/firewall.rules
```

2. 关闭防火墙

```
[root@cdh01 home]# systemctl disable firewalld.service 
Removed symlink /etc/systemd/system/multi-user.target.wants/firewalld.service.
Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.
[root@cdh01 home]# systemctl stop firewalld.service
```

### 配置SELinux模式

1. 检查SELinux状态

```
getenforce
```

2. 如果返回值是`Permissive`或者是`Disabled`可以跳到此步骤
3. 修改文件 `/etc/setlinux/config`，将`SELINUX=enforcing`改成`SELINUX=permissive`
4. 重启系统或者执行`setenforce 0`
5. 安装完CDH可以将配置改回去`enforcing`并执行`setenforce 1`

### 配置NTP

1. 安装`ntp`， 执行	`yum install ntp`
2. 修改`/etc/ntp.conf`

```
server 0.centos.pool.ntp.org
server 1.centos.pool.ntp.org
server 2.centos.pool.ntp.org
```

3. 启动`ntpd`

```
systemctl start ntpd
```

4. 配置`ntpd`为服务随开机启动

```
systemctl enable ntpd
```

5. 同步系统时间

```
ntpdate -u 0.pool.ntp.org
```

6. 同步硬件时间

```
hwclock --systohc
```



### 在Huc的机器上安装python2.7

### Impala依赖







## 【*】配置yum私有仓库

1. 安装httpd

```
yum -y install httpd
```

2. 修改配置`/etc/httpd/conf/httpd.conf` 将`AddType application/x-gzip .gz .tgz`修改为`AddType application/x-gzip .gz .tgz .parcel`

3. 启动httpd服务 `systemctl start httpd`
4. 将httpd设为自启 `systemctl enable httpd`

```
[root@cdh01 conf]# systemctl status httpd
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2022-02-14 23:48:50 PST; 1 day 18h ago
     Docs: man:httpd(8)
           man:apachectl(8)
 Main PID: 1181 (httpd)
   Status: "Total requests: 10; Current requests/sec: 0; Current traffic:   0 B/sec"
    Tasks: 9
   CGroup: /system.slice/httpd.service
           ├─ 1181 /usr/sbin/httpd -DFOREGROUND
           ├─ 1253 /usr/sbin/httpd -DFOREGROUND
           ├─ 1255 /usr/sbin/httpd -DFOREGROUND
           ├─ 1258 /usr/sbin/httpd -DFOREGROUND
           ├─ 1259 /usr/sbin/httpd -DFOREGROUND
           ├─ 1260 /usr/sbin/httpd -DFOREGROUND
           ├─48312 /usr/sbin/httpd -DFOREGROUND
           ├─48313 /usr/sbin/httpd -DFOREGROUND
           └─48314 /usr/sbin/httpd -DFOREGROUND

Feb 14 23:48:50 cdh01 systemd[1]: Starting The Apache HTTP Server...
Feb 14 23:48:50 cdh01 httpd[1181]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 192.168.43.129. Set the 'ServerName' directive globally to suppress this message
Feb 14 23:48:50 cdh01 systemd[1]: Started The Apache HTTP Server.
```



## **安装包列表**

```
[cdh@localhost 6.3.1]$ ls -lh
total 1.4G
-rwxrwxr-x. 1 cdh  cdh   14K Feb 14 17:59 allkeys.asc
-rwxrwxr-x. 1 cdh  cdh   10M Feb 14 18:01 cloudera-manager-agent-6.3.1-1466458.el7.x86_64.rpm
-rwxrwxr-x. 1 cdh  cdh  1.2G Feb 14 18:08 cloudera-manager-daemons-6.3.1-1466458.el7.x86_64.rpm
-rwxrwxr-x. 1 cdh  cdh   12K Feb 14 18:09 cloudera-manager-server-6.3.1-1466458.el7.x86_64.rpm
-rwxrwxr-x. 1 cdh  cdh   11K Feb 14 18:09 cloudera-manager-server-db-2-6.3.1-1466458.el7.x86_64.rpm
-rwxrwxr-x. 1 cdh  cdh   14M Feb 14 18:10 enterprise-debuginfo-6.3.1-1466458.el7.x86_64.rpm
-rwxrwxr-x. 1 cdh  cdh  177M Feb 14 18:14 oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm
drwxr-xr-x. 2 root root 4.0K Feb 14 18:56 repodata
-rwxrwxr-x. 1 cdh  cdh  3.1K Feb 14 18:10 RPM-GPG-KEY-cloudera

```



## 使用s3作为yum仓库

```
cat /etc/yum.repos.d/cloudera-manager.repo

[cloudera-manager]
baseurl = {host}/cdh/6.3.1
gpgkey = {host}/cdh/6.3.1/RPM-GPG-KEY-cloudera
enabled = 1
# 不check了，网上找的gpgkey好像不太对
gpgcheck = 0
autorefresh = 1
type = rpm-md
name = cloudera-manager
#repo_gpgcheck = 1

yum clean all
yum repolist

```



## 安装oraclejdk

```
[root@localhost 6.3.1]# yum install oracle-j2sdk1.8
Loaded plugins: fastestmirror, langpacks
Loading mirror speeds from cached hostfile
 * base: mirror.hostlink.com.hk
 * extras: mirror.hostlink.com.hk
 * updates: mirror.hostlink.com.hk
Resolving Dependencies
--> Running transaction check
---> Package oracle-j2sdk1.8.x86_64 0:1.8.0+update181-1 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

===================================================================================================================================================================================================================
 Package                                             Arch                                       Version                                                 Repository                                            Size
===================================================================================================================================================================================================================
Installing:
 oracle-j2sdk1.8                                     x86_64                                     1.8.0+update181-1                                       cloudera-manager                                     176 M

Transaction Summary
===================================================================================================================================================================================================================
Install  1 Package

Total download size: 176 M
Installed size: 364 M
Is this ok [y/d/N]: y
Downloading packages:
oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm                                                                                                                                                | 176 MB  00:00:07     
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : oracle-j2sdk1.8-1.8.0+update181-1.x86_64                                                                                                                                                        1/1 
  Verifying  : oracle-j2sdk1.8-1.8.0+update181-1.x86_64                                                                                                                                                        1/1 

Installed:
  oracle-j2sdk1.8.x86_64 0:1.8.0+update181-1                                                                                                                                                                       

Complete!

```


## 安装cloudera Manager Server

```
[root@localhost 6.3.1]# yum install cloudera-manager-daemons cloudera-manager-agent cloudera-manager-server -y
Loaded plugins: fastestmirror, langpacks
Loading mirror speeds from cached hostfile
 * base: mirror.hostlink.com.hk
 * extras: mirror.hostlink.com.hk
 * updates: mirror.hostlink.com.hk
Resolving Dependencies
--> Running transaction check
---> Package cloudera-manager-agent.x86_64 0:6.3.1-1466458.el7 will be installed
cloudera-manager/filelists_db                                                                                                                                                               | 118 kB  00:00:00     
--> Processing Dependency: python-psycopg2 for package: cloudera-manager-agent-6.3.1-1466458.el7.x86_64
base/7/x86_64/filelists_db                                                                                                                                                                  | 7.2 MB  00:00:03     
--> Processing Dependency: openssl-devel for package: cloudera-manager-agent-6.3.1-1466458.el7.x86_64
updates/7/x86_64/filelists_db                                                                                                                                                               | 7.4 MB  00:00:04     
--> Processing Dependency: mod_ssl for package: cloudera-manager-agent-6.3.1-1466458.el7.x86_64
--> Processing Dependency: MySQL-python for package: cloudera-manager-agent-6.3.1-1466458.el7.x86_64
--> Processing Dependency: /lib/lsb/init-functions for package: cloudera-manager-agent-6.3.1-1466458.el7.x86_64
extras/7/x86_64/filelists_db                                                                                                                                                                | 259 kB  00:00:00     
--> Processing Dependency: libpq.so.5()(64bit) for package: cloudera-manager-agent-6.3.1-1466458.el7.x86_64
---> Package cloudera-manager-daemons.x86_64 0:6.3.1-1466458.el7 will be installed
---> Package cloudera-manager-server.x86_64 0:6.3.1-1466458.el7 will be installed
--> Running transaction check
---> Package MySQL-python.x86_64 0:1.2.5-1.el7 will be installed
---> Package mod_ssl.x86_64 1:2.4.6-97.el7.centos.4 will be installed
---> Package openssl-devel.x86_64 1:1.0.2k-24.el7_9 will be installed
--> Processing Dependency: openssl-libs(x86-64) = 1:1.0.2k-24.el7_9 for package: 1:openssl-devel-1.0.2k-24.el7_9.x86_64
--> Processing Dependency: zlib-devel(x86-64) for package: 1:openssl-devel-1.0.2k-24.el7_9.x86_64
--> Processing Dependency: krb5-devel(x86-64) for package: 1:openssl-devel-1.0.2k-24.el7_9.x86_64
---> Package postgresql-libs.x86_64 0:9.2.24-7.el7_9 will be installed
---> Package python-psycopg2.x86_64 0:2.5.1-4.el7 will be installed
---> Package redhat-lsb-core.x86_64 0:4.1-27.el7.centos.1 will be installed
--> Processing Dependency: redhat-lsb-submod-security(x86-64) = 4.1-27.el7.centos.1 for package: redhat-lsb-core-4.1-27.el7.centos.1.x86_64
--> Processing Dependency: spax for package: redhat-lsb-core-4.1-27.el7.centos.1.x86_64
--> Processing Dependency: /usr/bin/m4 for package: redhat-lsb-core-4.1-27.el7.centos.1.x86_64
--> Running transaction check
---> Package krb5-devel.x86_64 0:1.15.1-51.el7_9 will be installed
--> Processing Dependency: libkadm5(x86-64) = 1.15.1-51.el7_9 for package: krb5-devel-1.15.1-51.el7_9.x86_64
--> Processing Dependency: krb5-libs(x86-64) = 1.15.1-51.el7_9 for package: krb5-devel-1.15.1-51.el7_9.x86_64
--> Processing Dependency: libverto-devel for package: krb5-devel-1.15.1-51.el7_9.x86_64
--> Processing Dependency: libselinux-devel for package: krb5-devel-1.15.1-51.el7_9.x86_64
--> Processing Dependency: libcom_err-devel for package: krb5-devel-1.15.1-51.el7_9.x86_64
--> Processing Dependency: keyutils-libs-devel for package: krb5-devel-1.15.1-51.el7_9.x86_64
---> Package m4.x86_64 0:1.4.16-10.el7 will be installed
---> Package openssl-libs.x86_64 1:1.0.2k-19.el7 will be updated
--> Processing Dependency: openssl-libs(x86-64) = 1:1.0.2k-19.el7 for package: 1:openssl-1.0.2k-19.el7.x86_64
---> Package openssl-libs.x86_64 1:1.0.2k-24.el7_9 will be an update
---> Package redhat-lsb-submod-security.x86_64 0:4.1-27.el7.centos.1 will be installed
---> Package spax.x86_64 0:1.5.2-13.el7 will be installed
---> Package zlib-devel.x86_64 0:1.2.7-19.el7_9 will be installed
--> Processing Dependency: zlib = 1.2.7-19.el7_9 for package: zlib-devel-1.2.7-19.el7_9.x86_64
--> Running transaction check
---> Package keyutils-libs-devel.x86_64 0:1.5.8-3.el7 will be installed
---> Package krb5-libs.x86_64 0:1.15.1-50.el7 will be updated
--> Processing Dependency: krb5-libs(x86-64) = 1.15.1-50.el7 for package: krb5-workstation-1.15.1-50.el7.x86_64
---> Package krb5-libs.x86_64 0:1.15.1-51.el7_9 will be an update
---> Package libcom_err-devel.x86_64 0:1.42.9-19.el7 will be installed
---> Package libkadm5.x86_64 0:1.15.1-50.el7 will be updated
---> Package libkadm5.x86_64 0:1.15.1-51.el7_9 will be an update
---> Package libselinux-devel.x86_64 0:2.5-15.el7 will be installed
--> Processing Dependency: libsepol-devel(x86-64) >= 2.5-10 for package: libselinux-devel-2.5-15.el7.x86_64
--> Processing Dependency: pkgconfig(libsepol) for package: libselinux-devel-2.5-15.el7.x86_64
--> Processing Dependency: pkgconfig(libpcre) for package: libselinux-devel-2.5-15.el7.x86_64
---> Package libverto-devel.x86_64 0:0.2.5-4.el7 will be installed
---> Package openssl.x86_64 1:1.0.2k-19.el7 will be updated
---> Package openssl.x86_64 1:1.0.2k-24.el7_9 will be an update
---> Package zlib.x86_64 0:1.2.7-18.el7 will be updated
---> Package zlib.x86_64 0:1.2.7-19.el7_9 will be an update
--> Running transaction check
---> Package krb5-workstation.x86_64 0:1.15.1-50.el7 will be updated
---> Package krb5-workstation.x86_64 0:1.15.1-51.el7_9 will be an update
---> Package libsepol-devel.x86_64 0:2.5-10.el7 will be installed
---> Package pcre-devel.x86_64 0:8.32-17.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

===================================================================================================================================================================================================================
 Package                                                    Arch                                   Version                                                  Repository                                        Size
===================================================================================================================================================================================================================
Installing:
 cloudera-manager-agent                                     x86_64                                 6.3.1-1466458.el7                                        cloudera-manager                                  10 M
 cloudera-manager-daemons                                   x86_64                                 6.3.1-1466458.el7                                        cloudera-manager                                 1.1 G
 cloudera-manager-server                                    x86_64                                 6.3.1-1466458.el7                                        cloudera-manager                                  11 k
Installing for dependencies:
 MySQL-python                                               x86_64                                 1.2.5-1.el7                                              base                                              90 k
 keyutils-libs-devel                                        x86_64                                 1.5.8-3.el7                                              base                                              37 k
 krb5-devel                                                 x86_64                                 1.15.1-51.el7_9                                          updates                                          273 k
 libcom_err-devel                                           x86_64                                 1.42.9-19.el7                                            base                                              32 k
 libselinux-devel                                           x86_64                                 2.5-15.el7                                               base                                             187 k
 libsepol-devel                                             x86_64                                 2.5-10.el7                                               base                                              77 k
 libverto-devel                                             x86_64                                 0.2.5-4.el7                                              base                                              12 k
 m4                                                         x86_64                                 1.4.16-10.el7                                            base                                             256 k
 mod_ssl                                                    x86_64                                 1:2.4.6-97.el7.centos.4                                  updates                                          115 k
 openssl-devel                                              x86_64                                 1:1.0.2k-24.el7_9                                        updates                                          1.5 M
 pcre-devel                                                 x86_64                                 8.32-17.el7                                              base                                             480 k
 postgresql-libs                                            x86_64                                 9.2.24-7.el7_9                                           updates                                          235 k
 python-psycopg2                                            x86_64                                 2.5.1-4.el7                                              base                                             132 k
 redhat-lsb-core                                            x86_64                                 4.1-27.el7.centos.1                                      base                                              38 k
 redhat-lsb-submod-security                                 x86_64                                 4.1-27.el7.centos.1                                      base                                              15 k
 spax                                                       x86_64                                 1.5.2-13.el7                                             base                                             260 k
 zlib-devel                                                 x86_64                                 1.2.7-19.el7_9                                           updates                                           50 k
Updating for dependencies:
 krb5-libs                                                  x86_64                                 1.15.1-51.el7_9                                          updates                                          809 k
 krb5-workstation                                           x86_64                                 1.15.1-51.el7_9                                          updates                                          820 k
 libkadm5                                                   x86_64                                 1.15.1-51.el7_9                                          updates                                          179 k
 openssl                                                    x86_64                                 1:1.0.2k-24.el7_9                                        updates                                          494 k
 openssl-libs                                               x86_64                                 1:1.0.2k-24.el7_9                                        updates                                          1.2 M
 zlib                                                       x86_64                                 1.2.7-19.el7_9                                           updates                                           90 k

Transaction Summary
===================================================================================================================================================================================================================
Install  3 Packages (+17 Dependent packages)
Upgrade             (  6 Dependent packages)

Total download size: 1.1 G
Downloading packages:
No Presto metadata available for updates
(1/26): MySQL-python-1.2.5-1.el7.x86_64.rpm                                                                                                                                                 |  90 kB  00:00:00     
(2/26): cloudera-manager-agent-6.3.1-1466458.el7.x86_64.rpm                                                                                                                                 |  10 MB  00:00:00     
(3/26): cloudera-manager-server-6.3.1-1466458.el7.x86_64.rpm                                                                                                                                |  11 kB  00:00:00     
(4/26): keyutils-libs-devel-1.5.8-3.el7.x86_64.rpm                                                                                                                                          |  37 kB  00:00:00     
(5/26): libcom_err-devel-1.42.9-19.el7.x86_64.rpm                                                                                                                                           |  32 kB  00:00:00     
(6/26): krb5-libs-1.15.1-51.el7_9.x86_64.rpm                                                                                                                                                | 809 kB  00:00:00     
(7/26): krb5-devel-1.15.1-51.el7_9.x86_64.rpm                                                                                                                                               | 273 kB  00:00:00     
(8/26): libselinux-devel-2.5-15.el7.x86_64.rpm                                                                                                                                              | 187 kB  00:00:00     
(9/26): libverto-devel-0.2.5-4.el7.x86_64.rpm                                                                                                                                               |  12 kB  00:00:00     
(10/26): libsepol-devel-2.5-10.el7.x86_64.rpm                                                                                                                                               |  77 kB  00:00:00     
(11/26): m4-1.4.16-10.el7.x86_64.rpm                                                                                                                                                        | 256 kB  00:00:00     
(12/26): mod_ssl-2.4.6-97.el7.centos.4.x86_64.rpm                                                                                                                                           | 115 kB  00:00:00     
(13/26): openssl-1.0.2k-24.el7_9.x86_64.rpm                                                                                                                                                 | 494 kB  00:00:00     
(14/26): openssl-libs-1.0.2k-24.el7_9.x86_64.rpm                                                                                                                                            | 1.2 MB  00:00:01     
(15/26): openssl-devel-1.0.2k-24.el7_9.x86_64.rpm                                                                                                                                           | 1.5 MB  00:00:01     
(16/26): postgresql-libs-9.2.24-7.el7_9.x86_64.rpm                                                                                                                                          | 235 kB  00:00:00     
(17/26): pcre-devel-8.32-17.el7.x86_64.rpm                                                                                                                                                  | 480 kB  00:00:00     
(18/26): python-psycopg2-2.5.1-4.el7.x86_64.rpm                                                                                                                                             | 132 kB  00:00:00     
(19/26): redhat-lsb-core-4.1-27.el7.centos.1.x86_64.rpm                                                                                                                                     |  38 kB  00:00:00     
(20/26): redhat-lsb-submod-security-4.1-27.el7.centos.1.x86_64.rpm                                                                                                                          |  15 kB  00:00:00     
(21/26): zlib-1.2.7-19.el7_9.x86_64.rpm                                                                                                                                                     |  90 kB  00:00:00     
(22/26): zlib-devel-1.2.7-19.el7_9.x86_64.rpm                                                                                                                                               |  50 kB  00:00:00     
(23/26): spax-1.5.2-13.el7.x86_64.rpm                                                                                                                                                       | 260 kB  00:00:00     
(24/26): libkadm5-1.15.1-51.el7_9.x86_64.rpm                                                                                                                                                | 179 kB  00:00:02     
(25/26): krb5-workstation-1.15.1-51.el7_9.x86_64.rpm                                                                                                                                        | 820 kB  00:00:03     
(26/26): cloudera-manager-daemons-6.3.1-1466458.el7.x86_64.rpm                                                                                                                              | 1.1 GB  00:00:43     
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                                                                                               27 MB/s | 1.1 GB  00:00:43     
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Updating   : zlib-1.2.7-19.el7_9.x86_64                                                                                                                                                                     1/32 
  Updating   : 1:openssl-libs-1.0.2k-24.el7_9.x86_64                                                                                                                                                          2/32 
  Updating   : krb5-libs-1.15.1-51.el7_9.x86_64                                                                                                                                                               3/32 
  Installing : postgresql-libs-9.2.24-7.el7_9.x86_64                                                                                                                                                          4/32 
  Updating   : 1:openssl-1.0.2k-24.el7_9.x86_64                                                                                                                                                               5/32 
  Updating   : libkadm5-1.15.1-51.el7_9.x86_64                                                                                                                                                                6/32 
  Installing : cloudera-manager-daemons-6.3.1-1466458.el7.x86_64                                                                                                                                              7/32 
  Installing : 1:mod_ssl-2.4.6-97.el7.centos.4.x86_64                                                                                                                                                         8/32 
  Installing : python-psycopg2-2.5.1-4.el7.x86_64                                                                                                                                                             9/32 
  Installing : MySQL-python-1.2.5-1.el7.x86_64                                                                                                                                                               10/32 
  Installing : zlib-devel-1.2.7-19.el7_9.x86_64                                                                                                                                                              11/32 
  Installing : libcom_err-devel-1.42.9-19.el7.x86_64                                                                                                                                                         12/32 
  Installing : spax-1.5.2-13.el7.x86_64                                                                                                                                                                      13/32 
  Installing : keyutils-libs-devel-1.5.8-3.el7.x86_64                                                                                                                                                        14/32 
  Installing : libverto-devel-0.2.5-4.el7.x86_64                                                                                                                                                             15/32 
  Installing : libsepol-devel-2.5-10.el7.x86_64                                                                                                                                                              16/32 
  Installing : m4-1.4.16-10.el7.x86_64                                                                                                                                                                       17/32 
  Installing : pcre-devel-8.32-17.el7.x86_64                                                                                                                                                                 18/32 
  Installing : libselinux-devel-2.5-15.el7.x86_64                                                                                                                                                            19/32 
  Installing : krb5-devel-1.15.1-51.el7_9.x86_64                                                                                                                                                             20/32 
  Installing : 1:openssl-devel-1.0.2k-24.el7_9.x86_64                                                                                                                                                        21/32 
  Installing : redhat-lsb-submod-security-4.1-27.el7.centos.1.x86_64                                                                                                                                         22/32 
  Installing : redhat-lsb-core-4.1-27.el7.centos.1.x86_64                                                                                                                                                    23/32 
  Installing : cloudera-manager-agent-6.3.1-1466458.el7.x86_64                                                                                                                                               24/32 
Created symlink from /etc/systemd/system/multi-user.target.wants/cloudera-scm-agent.service to /usr/lib/systemd/system/cloudera-scm-agent.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/supervisord.service to /usr/lib/systemd/system/supervisord.service.
  Installing : cloudera-manager-server-6.3.1-1466458.el7.x86_64                                                                                                                                              25/32 
Created symlink from /etc/systemd/system/multi-user.target.wants/cloudera-scm-server.service to /usr/lib/systemd/system/cloudera-scm-server.service.
  Updating   : krb5-workstation-1.15.1-51.el7_9.x86_64                                                                                                                                                       26/32 
  Cleanup    : 1:openssl-1.0.2k-19.el7.x86_64                                                                                                                                                                27/32 
  Cleanup    : krb5-workstation-1.15.1-50.el7.x86_64                                                                                                                                                         28/32 
  Cleanup    : libkadm5-1.15.1-50.el7.x86_64                                                                                                                                                                 29/32 
  Cleanup    : 1:openssl-libs-1.0.2k-19.el7.x86_64                                                                                                                                                           30/32 
  Cleanup    : krb5-libs-1.15.1-50.el7.x86_64                                                                                                                                                                31/32 
  Cleanup    : zlib-1.2.7-18.el7.x86_64                                                                                                                                                                      32/32 
  Verifying  : libselinux-devel-2.5-15.el7.x86_64                                                                                                                                                             1/32 
  Verifying  : redhat-lsb-submod-security-4.1-27.el7.centos.1.x86_64                                                                                                                                          2/32 
  Verifying  : krb5-libs-1.15.1-51.el7_9.x86_64                                                                                                                                                               3/32 
  Verifying  : krb5-workstation-1.15.1-51.el7_9.x86_64                                                                                                                                                        4/32 
  Verifying  : MySQL-python-1.2.5-1.el7.x86_64                                                                                                                                                                5/32 
  Verifying  : pcre-devel-8.32-17.el7.x86_64                                                                                                                                                                  6/32 
  Verifying  : cloudera-manager-agent-6.3.1-1466458.el7.x86_64                                                                                                                                                7/32 
  Verifying  : zlib-1.2.7-19.el7_9.x86_64                                                                                                                                                                     8/32 
  Verifying  : m4-1.4.16-10.el7.x86_64                                                                                                                                                                        9/32 
  Verifying  : postgresql-libs-9.2.24-7.el7_9.x86_64                                                                                                                                                         10/32 
  Verifying  : libsepol-devel-2.5-10.el7.x86_64                                                                                                                                                              11/32 
  Verifying  : 1:openssl-devel-1.0.2k-24.el7_9.x86_64                                                                                                                                                        12/32 
  Verifying  : libverto-devel-0.2.5-4.el7.x86_64                                                                                                                                                             13/32 
  Verifying  : 1:openssl-libs-1.0.2k-24.el7_9.x86_64                                                                                                                                                         14/32 
  Verifying  : keyutils-libs-devel-1.5.8-3.el7.x86_64                                                                                                                                                        15/32 
  Verifying  : 1:openssl-1.0.2k-24.el7_9.x86_64                                                                                                                                                              16/32 
  Verifying  : python-psycopg2-2.5.1-4.el7.x86_64                                                                                                                                                            17/32 
  Verifying  : krb5-devel-1.15.1-51.el7_9.x86_64                                                                                                                                                             18/32 
  Verifying  : 1:mod_ssl-2.4.6-97.el7.centos.4.x86_64                                                                                                                                                        19/32 
  Verifying  : libkadm5-1.15.1-51.el7_9.x86_64                                                                                                                                                               20/32 
  Verifying  : cloudera-manager-daemons-6.3.1-1466458.el7.x86_64                                                                                                                                             21/32 
  Verifying  : cloudera-manager-server-6.3.1-1466458.el7.x86_64                                                                                                                                              22/32 
  Verifying  : spax-1.5.2-13.el7.x86_64                                                                                                                                                                      23/32 
  Verifying  : libcom_err-devel-1.42.9-19.el7.x86_64                                                                                                                                                         24/32 
  Verifying  : redhat-lsb-core-4.1-27.el7.centos.1.x86_64                                                                                                                                                    25/32 
  Verifying  : zlib-devel-1.2.7-19.el7_9.x86_64                                                                                                                                                              26/32 
  Verifying  : krb5-libs-1.15.1-50.el7.x86_64                                                                                                                                                                27/32 
  Verifying  : libkadm5-1.15.1-50.el7.x86_64                                                                                                                                                                 28/32 
  Verifying  : zlib-1.2.7-18.el7.x86_64                                                                                                                                                                      29/32 
  Verifying  : krb5-workstation-1.15.1-50.el7.x86_64                                                                                                                                                         30/32 
  Verifying  : 1:openssl-1.0.2k-19.el7.x86_64                                                                                                                                                                31/32 
  Verifying  : 1:openssl-libs-1.0.2k-19.el7.x86_64                                                                                                                                                           32/32 

Installed:
  cloudera-manager-agent.x86_64 0:6.3.1-1466458.el7                    cloudera-manager-daemons.x86_64 0:6.3.1-1466458.el7                    cloudera-manager-server.x86_64 0:6.3.1-1466458.el7                   

Dependency Installed:
  MySQL-python.x86_64 0:1.2.5-1.el7             keyutils-libs-devel.x86_64 0:1.5.8-3.el7            krb5-devel.x86_64 0:1.15.1-51.el7_9                            libcom_err-devel.x86_64 0:1.42.9-19.el7       
  libselinux-devel.x86_64 0:2.5-15.el7          libsepol-devel.x86_64 0:2.5-10.el7                  libverto-devel.x86_64 0:0.2.5-4.el7                            m4.x86_64 0:1.4.16-10.el7                     
  mod_ssl.x86_64 1:2.4.6-97.el7.centos.4        openssl-devel.x86_64 1:1.0.2k-24.el7_9              pcre-devel.x86_64 0:8.32-17.el7                                postgresql-libs.x86_64 0:9.2.24-7.el7_9       
  python-psycopg2.x86_64 0:2.5.1-4.el7          redhat-lsb-core.x86_64 0:4.1-27.el7.centos.1        redhat-lsb-submod-security.x86_64 0:4.1-27.el7.centos.1        spax.x86_64 0:1.5.2-13.el7                    
  zlib-devel.x86_64 0:1.2.7-19.el7_9           

Dependency Updated:
  krb5-libs.x86_64 0:1.15.1-51.el7_9      krb5-workstation.x86_64 0:1.15.1-51.el7_9      libkadm5.x86_64 0:1.15.1-51.el7_9      openssl.x86_64 1:1.0.2k-24.el7_9      openssl-libs.x86_64 1:1.0.2k-24.el7_9     
  zlib.x86_64 0:1.2.7-19.el7_9           

Complete!

```


```shell
[cdh@localhost 6.3.1]$ wget https://ceph-uat-gw.heygears.com:17482/public-bucket/cdh/6.3.1/cdh-6.3.1.tar.gz
[cdh@localhost 6.3.1]$ tar -zxvf cdh-6.3.1.tar.gz
[cdh@localhost 6.3.1]$ yum -y install httpd createrepo
[cdh@localhost 6.3.1]$ systemctl status httpd
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
     Docs: man:httpd(8)
           man:apachectl(8)
[cdh@localhost 6.3.1]$ systemctl start httpd
[cdh@localhost 6.3.1]$ systemctl enable httpd
Created symlink from /etc/systemd/system/multi-user.target.wants/httpd.service to /usr/lib/systemd/system/httpd.service.
[cdh@localhost 6.3.1]$ systemctl status httpd
● httpd.service - The Apache HTTP Server
   Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2022-02-14 18:54:46 PST; 18s ago
     Docs: man:httpd(8)
           man:apachectl(8)
 Main PID: 72070 (httpd)
   Status: "Total requests: 0; Current requests/sec: 0; Current traffic:   0 B/sec"
   CGroup: /system.slice/httpd.service
           ├─72070 /usr/sbin/httpd -DFOREGROUND
           ├─72073 /usr/sbin/httpd -DFOREGROUND
           ├─72074 /usr/sbin/httpd -DFOREGROUND
           ├─72075 /usr/sbin/httpd -DFOREGROUND
           ├─72076 /usr/sbin/httpd -DFOREGROUND
           └─72077 /usr/sbin/httpd -DFOREGROUND
[cdh@localhost 6.3.1]$ 

[cdh@localhost 6.3.1]$ createrepo .
Spawning worker 0 with 2 pkgs
Spawning worker 1 with 2 pkgs
Spawning worker 2 with 1 pkgs
Spawning worker 3 with 1 pkgs
Workers Finished
Saving Primary metadata
Saving file lists metadata
Saving other metadata
Generating sqlite DBs
Sqlite DBs complete 
[cdh@localhost 6.3.1]$ ls -lh
total 1.4G
-rwxrwxr-x. 1 cdh cdh  14K Feb 14 17:59 allkeys.asc
-rwxrwxr-x. 1 cdh cdh  10M Feb 14 18:01 cloudera-manager-agent-6.3.1-1466458.el7.x86_64.rpm
-rwxrwxr-x. 1 cdh cdh 1.2G Feb 14 18:08 cloudera-manager-daemons-6.3.1-1466458.el7.x86_64.rpm
-rwxrwxr-x. 1 cdh cdh  12K Feb 14 18:09 cloudera-manager-server-6.3.1-1466458.el7.x86_64.rpm
-rwxrwxr-x. 1 cdh cdh  11K Feb 14 18:09 cloudera-manager-server-db-2-6.3.1-1466458.el7.x86_64.rpm
-rwxrwxr-x. 1 cdh cdh  14M Feb 14 18:10 enterprise-debuginfo-6.3.1-1466458.el7.x86_64.rpm
-rwxrwxr-x. 1 cdh cdh 177M Feb 14 18:14 oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm
drwxrwxr-x. 2 cdh cdh 4.0K Feb 14 18:43 repodata
-rwxrwxr-x. 1 cdh cdh 3.1K Feb 14 18:10 RPM-GPG-KEY-cloudera
[cdh@localhost 6.3.1]$ ls -lh repodata/
total 272K
-rw-rw-r--. 1 cdh cdh 8.7K Feb 14 18:43 388f6b908f1003a2dbfec558abe682d150359c77c31e79f6613902c6353220eb-primary.sqlite.bz2
-rw-rw-r--. 1 cdh cdh 1012 Feb 14 18:43 477117da349e68be7e2fbd7dbf704b1904381953c9465d12799252f8e4e24bb0-other.sqlite.bz2
-rw-rw-r--. 1 cdh cdh 119K Feb 14 18:43 7ae2ca783be50887569cda6d4d4a6235b6240a0abaa05a7b94ad372d86d7e1cf-filelists.sqlite.bz2
-rw-rw-r--. 1 cdh cdh  533 Feb 14 18:43 972c67399bd6846c906ba5db88901fb548b60576c5556fa2216f703baf925f39-other.xml.gz
-rw-rw-r--. 1 cdh cdh 3.3K Feb 14 18:43 9795d55c92221ae6b442f5d9686b6335ff2ada6f113d2a3bd0d2000ace608476-primary.xml.gz
-rw-rw-r--. 1 cdh cdh 123K Feb 14 18:43 d3e795d4db6f02b1565662011690c8d73800a4e671936c3eea169d21e4ecbef0-filelists.xml.gz
-rw-rw-r--. 1 cdh cdh 3.0K Feb 14 18:43 repomd.xml

```

## 配置auto-tls

```
[root@localhost jdk1.8.0_181-cloudera]# JAVA_HOME=/usr/java/jdk1.8.0_181-cloudera /opt/cloudera/cm-agent/bin/certmanager setup --configure-services
INFO:root:Logging to /var/log/cloudera-scm-agent/certmanager.log
[root@localhost jdk1.8.0_181-cloudera]# cat /var/log/cloudera-scm-agent/certmanager.log
[14/Feb/2022 22:24:03 +0000] 76212 MainThread cert         INFO     SCM Certificate Manager
[14/Feb/2022 22:24:03 +0000] 76212 MainThread os_ops       INFO     Created directory /var/lib/cloudera-scm-server/certmanager None None 0o755
[14/Feb/2022 22:24:03 +0000] 76212 MainThread os_ops       INFO     Created directory /var/lib/cloudera-scm-server/certmanager/private cloudera-scm cloudera-scm 0o700
[14/Feb/2022 22:24:03 +0000] 76212 MainThread os_ops       INFO     Created directory /var/lib/cloudera-scm-server/certmanager/trust-store cloudera-scm cloudera-scm 0o755
[14/Feb/2022 22:24:03 +0000] 76212 MainThread os_ops       INFO     Created directory /var/lib/cloudera-scm-server/certmanager/hosts-key-store cloudera-scm cloudera-scm 0o700
[14/Feb/2022 22:24:03 +0000] 76212 MainThread os_ops       INFO     Created directory /var/lib/cloudera-scm-server/certmanager/CMCA cloudera-scm cloudera-scm 0o700
[14/Feb/2022 22:24:03 +0000] 76212 MainThread os_ops       INFO     Created directory /var/lib/cloudera-scm-server/certmanager/CMCA/ca-db cloudera-scm cloudera-scm 0o700
[14/Feb/2022 22:24:03 +0000] 76212 MainThread os_ops       INFO     Created directory /var/lib/cloudera-scm-server/certmanager/CMCA/private cloudera-scm cloudera-scm 0o700
[14/Feb/2022 22:24:03 +0000] 76212 MainThread os_ops       INFO     Created directory /var/lib/cloudera-scm-server/certmanager/CMCA/ca-db/newcerts cloudera-scm cloudera-scm 0o700
[14/Feb/2022 22:24:04 +0000] 76212 MainThread os_ops       INFO     Created directory /var/lib/cloudera-scm-server/certmanager/hosts-key-store/localhost.localdomain cloudera-scm cloudera-scm 0o755

```



## 配置数据库（mysql）

cloudera推荐的配置，要使用目前已在用的mysql还是重新安装一个新的呢？

```
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
transaction-isolation = READ-COMMITTED
# Disabling symbolic-links is recommended to prevent assorted security risks;
# to do so, uncomment this line:
symbolic-links = 0

key_buffer_size = 32M
max_allowed_packet = 16M
thread_stack = 256K
thread_cache_size = 64
query_cache_limit = 8M
query_cache_size = 64M
query_cache_type = 1

max_connections = 550
#expire_logs_days = 10
#max_binlog_size = 100M

#log_bin should be on a disk with enough free space.
#Replace '/var/lib/mysql/mysql_binary_log' with an appropriate path for your
#system and chown the specified folder to the mysql user.
log_bin=/var/lib/mysql/mysql_binary_log

#In later versions of MySQL, if you enable the binary log and do not set
#a server_id, MySQL will not start. The server_id must be unique within
#the replicating group.
server_id=1

binlog_format = mixed

read_buffer_size = 2M
read_rnd_buffer_size = 16M
sort_buffer_size = 8M
join_buffer_size = 8M

# InnoDB settings
innodb_file_per_table = 1
innodb_flush_log_at_trx_commit  = 2
innodb_log_buffer_size = 64M
innodb_buffer_pool_size = 4G
innodb_thread_concurrency = 8
innodb_flush_method = O_DIRECT
innodb_log_file_size = 512M

[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

sql_mode=STRICT_ALL_TABLES
```

## 下载mysql-jdbc

```
wget https://ceph-uat-gw.heygears.com:17482/public-bucket/cdh/6.3.1/mysql-connector-java-5.1.46.tar.gz
tar -zxvf mysql-connector-java-5.1.46.tar.gz 
[root@localhost 6.3.1]# cd mysql-connector-java-5.1.46/
[root@localhost mysql-connector-java-5.1.46]# ls
build.xml  CHANGES  COPYING  mysql-connector-java-5.1.46-bin.jar  mysql-connector-java-5.1.46.jar  README  README.txt  src
[root@localhost mysql-connector-java-5.1.46]# sudo cp mysql-connector-java-5.1.46-bin.jar /usr/share/java/mysql-connector-java.jar

```

## 为cloudera software创建数据库

> reate databases and service accounts for components that require databases:
>
> - Cloudera Manager Server
> - Cloudera Management Service roles:
>   - Activity Monitor (if using the MapReduce service in a CDH 5 cluster)
>   - Reports Manager
> - Hue
> - Each Hive metastore
> - Sentry Server
> - Cloudera Navigator Audit Server
> - Cloudera Navigator Metadata Server
> - Oozie



```mysql
# CREATE DATABASE <database> DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
# GRANT ALL ON <database>.* TO '<user>'@'%' IDENTIFIED BY '<password>';

CREATE DATABASE scm DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON scm.* TO 'scm'@'%' IDENTIFIED BY 'scm';

CREATE DATABASE amon DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON amon.* TO 'amon'@'%' IDENTIFIED BY 'amon';

CREATE DATABASE rman DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON rman.* TO 'rman'@'%' IDENTIFIED BY 'rman';

CREATE DATABASE hue DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON hue.* TO 'hue'@'%' IDENTIFIED BY 'hue';

CREATE DATABASE metastore DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON metastore.* TO 'hive'@'%' IDENTIFIED BY 'hive';

CREATE DATABASE sentry DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON sentry.* TO 'sentry'@'%' IDENTIFIED BY 'sentry';

CREATE DATABASE nav DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON nav.* TO 'nav'@'%' IDENTIFIED BY 'nav';

CREATE DATABASE navms DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON navms.* TO 'navms'@'%' IDENTIFIED BY 'navms';

CREATE DATABASE oozie DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON oozie.* TO 'oozie'@'%' IDENTIFIED BY 'oozie';
```

## 配置ClouderaManager数据库

https://docs.cloudera.com/documentation/enterprise/6/latest/topics/prepare_cm_database.html#cmig_topic_5_2

> --scm-host:  The hostname where the Cloudera Manager Server is installed. If the Cloudera Manager Server and the database are installed on the same host, do not use this option or the -h option.

```
[root@localhost schema]# sudo /opt/cloudera/cm/schema/scm_prepare_database.sh mysql -h mysql -P 3306 scm scm
Enter SCM password: 
JAVA_HOME=/usr/java/jdk1.8.0_181-cloudera
Verifying that we can write to /etc/cloudera-scm-server
Creating SCM configuration file in /etc/cloudera-scm-server
Executing:  /usr/java/jdk1.8.0_181-cloudera/bin/java -cp /usr/share/java/mysql-connector-java.jar:/usr/share/java/oracle-connector-java.jar:/usr/share/java/postgresql-connector-java.jar:/opt/cloudera/cm/schema/../lib/* com.cloudera.enterprise.dbutil.DbCommandExecutor /etc/cloudera-scm-server/db.properties com.cloudera.cmf.db.
Mon Feb 14 22:29:33 PST 2022 WARN: Establishing SSL connection without server's identity verification is not recommended. According to MySQL 5.5.45+, 5.6.26+ and 5.7.6+ requirements SSL connection must be established by default if explicit option isn't set. For compliance with existing applications not using SSL the verifyServerCertificate property is set to 'false'. You need either to explicitly disable SSL by setting useSSL=false, or set useSSL=true and provide truststore for server certificate verification.
[                          main] DbCommandExecutor              INFO  Successfully connected to database.
All done, your SCM database is configured correctly!
[root@localhost schema]# cat /etc/cloudera-scm-server/db.properties
# Auto-generated by scm_prepare_database.sh on Mon Feb 14 22:29:32 PST 2022
#
# For information describing how to configure the Cloudera Manager Server
# to connect to databases, see the "Cloudera Manager Installation Guide."
#
com.cloudera.cmf.db.type=mysql
com.cloudera.cmf.db.host=mysql:3306
com.cloudera.cmf.db.name=scm
com.cloudera.cmf.db.user=scm
com.cloudera.cmf.db.setupType=EXTERNAL
com.cloudera.cmf.db.password=scm
```

## 安装CDH以及其他软件

```
[root@localhost schema]# systemctl status cloudera-scm-server.service 
● cloudera-scm-server.service - Cloudera CM Server Service
   Loaded: loaded (/usr/lib/systemd/system/cloudera-scm-server.service; enabled; vendor preset: disabled)
   Active: inactive (dead)
[root@localhost schema]# systemctl start cloudera-scm-server.service 
[root@localhost schema]# systemctl status cloudera-scm-server.service 
● cloudera-scm-server.service - Cloudera CM Server Service
   Loaded: loaded (/usr/lib/systemd/system/cloudera-scm-server.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2022-02-14 22:36:35 PST; 2s ago
  Process: 76538 ExecStartPre=/opt/cloudera/cm/bin/cm-server-pre (code=exited, status=0/SUCCESS)
 Main PID: 76545 (java)
    Tasks: 17
   CGroup: /system.slice/cloudera-scm-server.service
           └─76545 /usr/java/jdk1.8.0_181-cloudera/bin/java -cp .:/usr/share/java/mysql-connector-java.jar:/usr/share/java/oracle-connector-java.jar:/usr/share/java/postgresql-connector-java.jar:lib/* -server...

Feb 14 22:36:35 localhost.localdomain systemd[1]: Starting Cloudera CM Server Service...
Feb 14 22:36:35 localhost.localdomain systemd[1]: Started Cloudera CM Server Service.
Feb 14 22:36:35 localhost.localdomain cm-server[76545]: JAVA_HOME=/usr/java/jdk1.8.0_181-cloudera
Feb 14 22:36:36 localhost.localdomain cm-server[76545]: Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=256m; support was removed in 8.0
Feb 14 22:36:36 localhost.localdomain cm-server[76545]: Setting feature flag for AUTO_TLS via system env to true
Feb 14 22:36:36 localhost.localdomain cm-server[76545]: ERROR StatusLogger No log4j2 configuration file found. Using default configuration: logging only errors to the console. Set system property...tion logging.
Hint: Some lines were ellipsized, use -l to show in full.

```

1. 启动Cloudera Manager Server `sudo systemctl start cloudera-scm-server`
2. 查看日志 `tail -f /var/log/cloudera-scm-server/cloudera-scm-server.log`
3. 



```
cdh01.cdhserver.com cdh02.cdhserver.com cdh03.cdhserver.com
```



# troubleshooting

## CDH6在安装agent时，提示安装失败 无法接收 Agent 发出的检测信号

https://segmentfault.com/q/1010000016619711

解决办法： 只能说取消`auto-tls`功能

https://community.cloudera.com/t5/Support-Questions/Disable-remove-auto-TLS-certificates-and-create-self-signed/td-p/302507

## 安装parcel， 主机运行状况不良

https://blog.csdn.net/Post_Yuan/article/details/79101618

解决办法：

1. 删除cm_guid文件, `rm -rf /var/lib/cloudera-scm-agent/cm_guid`
2. 重启agent， `systemctl restart cloudera-scm-agent.service`

## 卸载cloudera-manager

https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/cm_ig_uninstall_cm.html#cmig_topic_18_1_1

## Inspect Cluster

![image-20220217150219962](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20220217150219962.png)

## Able to find the Database server, but not the specified database. Please check if the database name is correct and make sure that the user can access the database.

https://blog.csdn.net/u010886217/article/details/85345999



