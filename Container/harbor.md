private registry
----------------

docker默认拒绝使用http访问registry，为了快速创建私有registry，docker专门提供的一个程序包：docker-distribution，且在dockerhub上也有registry的容器仓库，运行registry容器必须为其定义一个存储卷，且此存储卷应该时共享存储。
```shell
# yum install  docker-registry -y	#实际安装过程安装的还是docker-distribution包
# rpm -ql docker-distribution		#查看配置文件路径、镜像存储路径、服务名称
# vim /etc/docker-distribution/registry/config.yml	#修改镜像存储路径
# systemctl start docker-distribution.service	#启动服务，默认使用5000端口
```
直接使用本地地址是可以上传镜像的，但如果非本地地址，又不是https协议，则需要修改/etc/docker/daemon.json文件
```shell
"insecure-registries": ["10.250.1.11:5000"]	#添加此行，IP地址可以更改为域名
```
重启docker服务后，修改镜像仓库名称，上传镜像即可

harbor
------

harbor的项目代码托管在github上，在安装harbor之前需要确认主机上已经安装了docker和docker-compose，确切的版本要求github上面也明确[要求](https://github.com/goharbor/harbor)了
[harbor配置文件信息](https://goharbor.io/docs/2.0.0/install-config/configure-yml-file/)<br />
配置文件中分为必要配置和可选配置，必要配置至少要修改主机名，可以选择性修改harbor的管理员密码和数据库密码，如果没有为https配置证书，那https的内容需要注释，否则会报错<br />
[harbor脚本安装](https://goharbor.io/docs/2.0.0/install-config/run-installer-script/)<br />
直接运行安装脚本即可，安装完成后查看端口是否开放
```diff
- harbor.yml文件中开放的端口号必须要与/etc/docker/daemon.json文件中的端口号相同，否则即便安装完成也无法登录web
```