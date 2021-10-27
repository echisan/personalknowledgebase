# 使用静态IP

修改配置文件

```shell
ubuntu@node02:~$ cat /etc/network/interfaces
auto enp0s8
iface enp0s8 inet static
address 192.168.56.7
netmask 255.255.255.0
```

启用网卡
```shell
ubuntu@node02:~$ sudo ifup enp0s8
```

