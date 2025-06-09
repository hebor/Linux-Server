docker镜像管理基础
=================

Docker镜像含有容器所需要的文件系统及其内容，且采用分层构建机制，最底层为bootfs，其次为rootfs
- bootfs：用于系统引导的文件系统，包括bootloader和kernel，容器启动完成后会被卸载以节约内存资源
- rootfs：位于bootfs之上，表现为docker容器的根文件系统
	* 传统模式中，系统启动时，内核挂载rootfs时首先将其挂载为"只读"模式，完整性自检完成后重新挂载为读写模式
	* docker中，rootfs由内核挂载为"只读"模式，而后通过"联合挂载"技术额外挂载一个"可写"层
位于下层的镜像称为父镜像，最底层的称为基础镜像(base image)

#### Docker Registry分类
- Sponsor Registry：第三方的registry，供客户与Dockers社区使用
- Mirror Registry：第三方的registry，只让用户可用
- Vendor Registry：由发布Dokcer镜像的供应商提供的registry
- Private Registry：通过设有防火墙和额外的安全层的私有实体提供的registry

#### repository
Repository
- 由特定的docker镜像的所有迭代版本组成的镜像仓库
- 一个registry中可存在多个repository
	- repository可分为"顶层仓库"和"用户仓库"
	- 用户仓库名称格式为"用户名/仓库名"
- 每个仓库可以包含多个tag，每个tag对应一个镜像

### 制作镜像
制作镜像的三种方式：
* Dockerfile
* 基于容器制作
* Docker Hub automated builds

示例：基于容器制作镜像
```shell
# docker container run --name box -it busybox:latest /bin/sh    #运行一个容器

#在容器中做出修改
/ # mkdir -p /data/html
/ # echo 'Busybox Server.' > /data/html/index.html

# docker container commit -a "hebo1248@163.com" -p box    #以容器box为模板制作镜像，a参数表示指定作者，p参数表示制作镜像时短暂暂停容器的运行
# docker image tag 0b7a893fa4f5 hbdbenben/student:box1    #为镜像打标签
# docker rmi 0b7a893fa4f5    #删除标签
```
```diff
- 基于容器制作镜像时并不是制作了一个完整的镜像，而是将容器的读写层的修改内容单独制作成了一个镜像
```
注：
	1. 制作镜像时不指定仓库和标签，则此镜像仓库和tag都为空，这种镜像也被称为虚悬镜像
	2. tag选项也可以添加标签，可以为一个镜像打上多个标签，类似硬链接，删除标签时不会删除镜像

新镜像制作成功了，但是使用镜像启动容器时的默认命令却没有更改，使用以下命令查看默认启动的命令：
```shell
# docker inspect f0b02e9d092d    #查看Cmd
```
commit选项自带了一个参数用于更改默认启动命令
```shell
# docker container commit -a "hebo1248@163.com" -c 'CMD ["/bin/httpd","-f","-h","/data/html"]' -p box hbdbenben/student:test
```
注：参数顺序不可错

### 上传镜像
上传镜像时需要指定registry地址，如果没有指定，则默认使用dockerhub，也就是docker.io，上传镜像的前提是打标签时要注意用户名和仓库名必须与dockerhub上的相对应，比如hbdbenben/student:test，hbdbenben表示用户名，每个人的用户名不一样，student表示仓库名，dockerhub上用户名是固定的，则必须要创建一个student仓库，本地镜像才能上传成功 <br />
示例：上传镜像
```shell
# docker push hbdbenben/student:test
```
这里使用了默认的dockerhub，所以完整的标签名应该是docker.io/hbdbenben/student:test，上传的完整路径也应该是
```shell
# docker push docker.io/hbdbenben/student:test
```
如果使用registry不是dockerhub，那在打标签和上传镜像时就应该明确指出使用的registry地址，比如使用阿里云的registry
```shell
# docker push {阿里云的registry地址}/hbdbenben/student
```
上传镜像时如何不指定tag，那将会上传整个repository的镜像

### 镜像的导入和导出
示例：镜像打包
```shell
# docker image save -o images.gz 02bdbb9f44a 8a2fb25a19f    #将多个镜像打包到一起
```
注：使用image id打包，解包后时不带有仓库名和标签的，也就是说打开后是一个虚悬镜像。
也可以仅指定单个镜像打包，打包完成后使用scp或其他命令到其他主机上解包即可 <br />
示例：镜像解包
```shell
# docker image load -i images.gz    #镜像解包
```
```diff
- 注：使用save打包镜像时仅打包了容器的读写层镜像，所以导入镜像之前，主机上需要下载好对应的base image，否则直接运行解包的镜像时还是会先pull基础镜像
```
