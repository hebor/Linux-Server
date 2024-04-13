# Nmap

网络发现和安全审计工具，常被用于网络扫描、端口扫描、主机探测、操作系统侦测等，其图形化界面是Zenmap

Nmap常用参数

```shell
-T4：指定扫描过程的级别，级别越高扫描速度越快，但也越容易被防火墙或IDS屏蔽，一般推荐使用T4级别
-sn：只进行主机发现，不进行端口扫描
-O：只进行系统版本扫描
-sV：进行服务版本扫描
-p：扫描指定端口
-sS：发送SYN包扫描
-PO：不进行ping扫描
--script：指定脚本扫描
```

Nmap常见的6中端口状态

```shell
open：开放的。表示应用程序正在监听该端口的连接，外部可以访问
filtered：被过滤的。表示端口被防火墙或其他网络设备阻止，不能访问
closed：关闭的。表示目标主机未开启该端口
unfiltered：未被过滤的。表示nmap无法确定端口状态，需要进一步探测
open/filtered：开放或未被过滤的，Nmap不能识别
closed/filtered：关闭或未被过滤的，Nmap不能识别
```

Nmap使用示例

```shell
# 1.扫描指定主机，默认会扫描出该主机打开的所有端口，也可以通过域名扫描
nmap 10.66.1.103

# 2.扫描网段，扫描网段内的存活主机
nmap 10.66.1.0/24

# 3.主机探测
nmap -T4 -sn 10.66.1.103
Nmap scan report for 10.66.1.103
Host is up (0.0010s latency).		# 主机UP

# 4.版本侦测
nmap 10.66.1.103 -O -sV -T4
PORT     STATE  SERVICE VERSION
22/tcp   open   ssh     OpenSSH 7.4 (protocol 2.0)
111/tcp  open   rpcbind 2-4 (RPC #100000)
2049/tcp closed nfs
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.10 - 4.11

# 5.端口范围扫描
nmap 10.66.1.103 -p 0-1000

# 6.跳过ping测试直接扫描端口
nmap 10.66.1.103 -p 0-1000 -PO

# 7.常用脚本
nmap --script=vuln 10.66.1.103		# 扫描目标主机是否存在常见的漏洞
nmap --script=exploit 10.66.1.103		# 利用已知漏洞入侵系统，没有任何回显代表入侵失败
```