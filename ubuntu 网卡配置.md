**从ubuntu 17.10 版本开始,就已经放弃在`/etc/network/interface`文件中固定IP的配置,即便配置也不会生效,而是改成`netplan`的方式写在`/etc/netplan/`目录下的yaml配置文件中**

示例:17.10版本以前的网卡配置

```shell
# 静态
auto enp0s25
iface enp0s25 inet static
address 10.0.0.5
netmask 255.255.255.0
gateway 10.0.0.254
dns-nameserver 114.114.114.114

# 动态
auto enp0s25
iface enp0s25 inet dhcp
```

示例:17.10版本以后的网卡配置

```shell
network: 
  ethernets: 
    wlp3s0: 
      addresses: 
      - 192.168.43.164/24
      dhcp4: no
      gateway4: 192.168.43.1
      nameservers: 
        addresses: [114.114.114.114,8.8.8.8]
  version: 2
```

配置文件写入后,使用`sudo netplan apply`使配置文件生效