docker 存储卷
============
docker镜像由多个只读层联合构成，启动一个容器，docker会加载只读镜像，且在镜像的最上层添加一个读写层，用于保存对容器的读写操作。而对只读镜像的操作则需要通过**写时复制**机制实现
```diff
+ 写时复制(COW)：如果运行中的容器修改了一个已存在的文件，那该文件将会从只读层复制到读写层，该文件的只读版本仍然存在，但会隐藏起来，转而只显示修改后的文件
```
且不论容器读写层数据持久化问题，写时复制的I/O机制在效率上就比较低下，如果遇到对I/O要求较高的服务，则需要使用存储卷的方式实现I/O读写

存储卷
-----
存储卷的实现则是宿主机为容器提供一个目录，容器将宿主机的目录挂载在自己的文件系统上，在容器中对此目录的读写会绕过容器文件系统的限制，直接对宿主机目录进行读写，宿主机上也可以通过此目录为容器提供共享数据。 <br />
通过存储卷的特性，还可以在容器间共享存储数据，并且在删除容器时，保存在存储卷中的数据默认是不会删除的，再次创建容器时引用此存储卷，可以较大程度上还原原本的容器。如果此存储卷所在的后端存储是一个共享存储，那任意一台连接此后端存储的主机都能够再次调度此容器。

volume types
------------
### bind mount volume
绑定挂载卷需要在容器和宿主机上都要手动指定一个目录，使两个已知路径建立关联关系 <br />
示例：
```shell
# docker container run --name b2 --rm -it -v /mnt/docker/valumes:/data busybox
	将宿主机的/mnt/docker/valumes路径绑定到容器的/data，指定的宿主机路径如果不存在，此路径会自动创建
```

### docker-managed volume
docker管理卷，只需要在容器上指定一个目录，而宿主机上则会由docker引擎新建一个随机目录或使用一个已存在的目录与容器的存储卷路径建立关联关系。使用docker管理卷时需要注意，如果删除了容器，那么下一次想引用此存储卷时，则需要用绑定挂载卷的方式引用，否则docker引擎会创建一个新的目录作为存储卷路径 <br />
示例：
```shell
# docker container run --name b1 -it -v /data busybox
	容器上指定/data目录为存储卷路径
# docker inspect b1
	查看容器详细信息，查看宿主机上的存储卷路径：Mounts.Source，容器上的存储卷路径则是：Config.Volumes
```

补充
----
* GO语言模板过滤
* 容器共享存储卷

### 使用GO语言模板过滤详细信息
示例：查看挂载信息
```shell
# docker inspect -f {{.Mounts}} b2
	-f：使用GO模板语言格式
	.Mounts：.表示从根目录开始，.Mounts表示根目录下的Mounts项

# docker inspect -f {{.NetworkSettings.IPAddress}} b2
	过滤容器的IP地址
```

### 容器共享存储卷
创建容器时可以将宿主机上的存储卷路径给多个容器使用，也就可以做到多容器共享数据，创建容器时可以直接连接到其他容器在宿主机上的存储卷 <br />
示例：
```shell
# docker run --name init -it --rm -v /mnt/docker/valumes:/data/init/volume busybox
	启动一个底层容器，并指定挂载卷
# docker container run --name web --network container:init --volumes-from init -it --rm busybox
	此容器共享init容器的网络和存储卷
```