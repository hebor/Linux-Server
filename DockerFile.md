# DockerFile

**容器配置文件的修改**

大多数业务场景下默认的镜像配置无法符合实际需求，因此就必须要修改默认的镜像配置，此前学习的内容大多是启动容器后对配置文件进行修改，修改配置后重启容器，这个过程显得过于繁琐。当然，也可以使用自制镜像（基于容器制作）或存储卷的方式修改或替换配置文件，但这种方式既不灵活又无法用于批量操作

docker在配置文件上这个问题上的解决方案是，构建镜像时，以变量的方式修改配置文件，启动容器时，docker会使用一个其他程序先替换容器的主进程，这个程序会将用户的传参替换到配置文件中的变量，传参完后，这个程序会通过exec命令再次启动容器的主进程。如果用户没有传参，则使用镜像配置文件的默认参数

以nginx镜像为例，通过nginx镜像启动容器时，docker会使用一个其他程序读取用户为nginx的配置文件传入的参数，此例中可以传入主机名、监听地址、html主目录，使用用户传入的参数替换掉配置文件中的变量后保存为新的配置文件，并通过此配置文件启动nginx容器

```shell
/etc/nginx/conf.d/server.conf
{
	server_name $NGX_SERVER_NAME;
	listen $NGX_IP:$NGINX_PORT;
	root $DOC_ROOT;
}
```

## DockerFile Format

DockerFile是另一种制作镜像的方式，DockerFile本质上只是一个包含构建镜像的指令合集的文本文件，与Linux命令行和Shell的关系相似。DockerFile文件中，"#"号开头表示注释，在整个Dockerfile文件中第一个非注释行必须以"FROM"开头，"FROM"将指定一个基础镜像，后续构建的镜像将基于这个基础镜像实现

DockerFile文件只有2种格式，一种是以“#”号开头的注释，另一种是指令，指令本身不区分大小写，但为了区分指令和其他参数符号，通常指令都使用大写，这并不是强制要求

**DockerFile文件格式上需要注意的几点**

1. 制作Dockerfile时必须在一个特定的工作目录下进行，且Dockerfile文件名首字母必须大写
2. Dockerfile文件中可以引用同级目录或子目录下的文件，但不能引用父目录的文件
3. 在工作目录下可以创建一个.dockeringore，写在此隐藏文件中的路径可以作为Dockerfile的黑名单，不被Dockefile引用。一般是一行一个文件路径，也可以使用通配符
4. 在Dockerfile中也可以执行shell命令，但这些shell命令必须是引用的底层镜像所包含的

## DockerFile构建指令集

指令集锚点：<a href="#from">FROM</a>、<a href="#maintainer">MAINTAINER</a>、<a href="#label">LABEL</a>、<a href="#copy">COPY</a>、<a href="#add">ADD</a>、<a href="#workdir">WORKDIR</a>、<a href="#volume">VOLUME</a>、<a href="#expose">EXPOSE</a>、<a href="#env">ENV</a>、<a href="#run">RUN</a>、<a href="#cmd">CMD</a>、<a href="#entrypoint">ENTRYPOINT</a>、<a href="#user">USER</a>

<span id="from">**FROM**</span>

FROM指令是最重要的一个指令且必须为Dockerfile文件第一个非注释行，用于为镜像文件构建过程中指定基准镜像，后续的指令运行基于此基准镜像提供的运行环境。基准镜像可以是任何可用镜像文件，默认情况下，docker build会在docker主机上查找指定的基础镜像文件，主机上不存在时会从Docker Hub Registry下载基础镜像，如果找不到指定的镜像，docker build会返回错误信息

语法格式：

```shell
FROM <repository>:[:<tag>] 或
FROM <repository>@<digest>
	digest：镜像的hash码
	repository：默认使用dockerhub的镜像，也可以指定别的registry
```

<span id="maintainer">**MAINTAINER（已废弃，被兼容）**</span>

用于让Dockerfile制作者提供本人信息，Dockerfile不限制MAINTAINER出现的位置，但推荐将其放在FROM指令之后 

语法格式：

```shell
MAINTAINER "hebor <hebo1248@163.com>"
```

<span id="label">**LABEL**</span>

LABEL为镜像指定元数据，LABEL使用键值对的表现形式，MAINTAINER可以通过LABEL实现。LABEL能够提供各种各样的key:value信息，MAINTAINER只是其中一项，所以LABEL比MAINTAINER具备更广泛的应用场景

语法格式：

```shell
LABEL <key>=<value> <key>=<value> ...
LABEL maintainer="hebo1248@163.com"
```

<span id="copy">**COPY**</span>

用于从宿主机复制文件到构建的新镜像文件中，源文件路径一般是相对路径，支持使用通配符，目标路径建议使用绝对路径，否则目标路径会以WORKDIR为跟路径。文件复制准则如下

1. 被复制的文件必须与Dockerfile处于同级或下级目录，不能是父目录中的文件
2. 如果被复制的是目录，则此目录下的所有文件及目录都会被复制，但此目录本身不会被复制
3. 如果指定了多个源文件或目录，或者在其中使用的通配符，那目标路径必须是一个目录，且必须以"/"结尾
4. 如果目标路径事先不存在，他将会被自动创建，包括父目录路径

语法格式：

```shell
COPY <src> ... <dest> 或
COPY ["<src>",... "<dest>"]    #通常源文件路径中存在空白字符时使用第二种格式
```

编写Dockerfile文件并进行测试

```shell
mkdir /srv/hebor	#新建工作目录
vim /srv/hebor/Dockerfile	#编写Dockerfile文件

# Description: first Dockerfile
FROM busybox:latest
#MAINTAINER "HeBor <hebo1248@163.com>"
LABEL maintainer="HeBor <hebo1248@163.com>"
COPY index.html /data/web/html/
COPY yum.repos.d /etc/yum.repos.d/

cp -r /etc/yum.repos.d/ /srv/hebor/	#宿主机拷贝目录到工作目录下
vim /srv/hebor/index.html	#新建宿主机文件
<h1>Busybox http server</h1>

docker build -t http:v0.1 /srv/hebor/	#构建镜像文件并打标签，构建镜像时必须指定工作目录
docker container run --rm http:v0.1 cat /data/web/html/index.html
	#启动容器，仅执行cat命令，退出删除容器。这种方式会改变镜像原本要运行的默认主程序，执行完cat命令后就会退出容器
```

在Dockerfile文件中，同一个指令可以出现多次多行，但每一条指令都会生成一个新的镜像层，所以在编写Dockerfile文件时应尽可能的精简文件内容

<span id="add">**ADD**</span>

ADD指令类似COPY指令，ADD支持使用tar文件和url路径，此指令似乎不支持使用"&&"语法。操作准则：
1. COPY的文件复制准则在ADD中一样适用
2. 如果源为url且目标路径不是目录，则源指定的url将被下载并直接重命名为目标文件；如果目标路径是目录，则源指定的url文件将会被下载到目标目录下
3. 如果源是一个本地系统的压缩文件，它将会被展开为解压后的目录；通过url获取到的压缩文件则不会自动展开
4. 有多个源目标或使用通配符时，目标路径必须是个目录，如果目标路径不以"/"结尾，则所有源文件的内容都会被直接写入目标文件

语法格式：

```shell
ADD <src> ... <dest> 或
ADD ["<src>",... "<dest>"]
```

<span id="workdir">**WORKDIR**</span>

为Dockerfile中所有的RUN、CMD、ENTRYPOINT、COPY和ADD指定工作目录。在Dockerfile中，WORKDIR指令可出现多次，其路径也可以为相对路径，不过相对路径是作用在此前一个WORKDIR指令指定的路径之上的。WORKDIR指令的每一次重新指定都只会影响在其之后的指令，另外，WORKDIR也可调用由ENV定义的变量。语法格式：

```shell
WORKDIR $PATH
WORKDIR /var/log
```

<span id="volume">**VOLUME**</span>

用于在镜像中创建一个挂载卷，但在Dockerfile中只能使用docker-managed volume，无法指定bind mount volume。语法格式：

```shell
VOLUME <mountpoint> 或
VOLUME ["<mountpoint>"]
```

<span id="expose">**EXPOSE**</span>

用于为容器打开指定要监听的端口，EXPOSE只能指定容器要开放的端口，使用随机高端口的方式映射到宿主机，不能直接指定映射到宿主机的端口。出于安全性考量，即便在镜像文件中有写入指定要暴露的端口，默认启动容器时指定的端口仍不会暴露，需要通过-P选项手动暴露端口，<protocol>用于指定传输协议，可为tcp和udp二选一，默认为tcp协议。语法格式：

```shell
EXPOSE <port>[/<protocol>] [<port>[/<protocol>] ...]
EXPOSE 11211/udp 11211/tcp
```

<span id="env">**ENV**</span>

用于为镜像定义所需要的环境变量，可被Dockerfile文件中位于其后的其他指令调用，例如ENV（嵌套调用）、ADD、COPY等，调用格式为$variable_name或${variable_name}。语法格式：

```shell
ENV <key> <value> 或    #此格式中，<key>之后的所有内容都被视为<value>的组成，所以一次只能设置一个变量
ENV "<key>=<value>"    #此格式可一次设置多个变量，每个变量通过键值对的方式表现为"<key>=<value>"，如果<value>中包含空格，可以用\转义或对其加引号进行标识，一个指令定义多个变量时也可以通过反斜线换行来展示的更具条理性
```

**docker run和docker build的关系**

Dockerfile示例文件

```shell
# Description： first Dockerfile
FROM busybox:latest
#MAINTAINER "HeBor <hebo1248@163.com>"
LABEL maintainer="hebor <hebo1248@163.com>"
ENV Doc_Root=/data/web/html/ \
	Web_Server_Package="nginx-1.25.2"
COPY index.html ${Doc_Root:-/data/web/html/}
COPY yum.repos.d /etc/yum.repos.d/
WORKDIR /usr/local/
ADD ${Web_Server_Package}.tar.gz ./nginx/
VOLUME /data/volume/
EXPOSE 80/tcp
```

执行`docker container run -e`和`docker build`的时候都是可以向变量传值的，但启动容器和镜像构建是两个不同的阶段，在Dockerfile文件中设置ENV环境变量，它将在`docker build`阶段执行，而在docker run时也能够执行变量替换，但`docker run`传参只是修改环境变量，它并不能修改`docker build`时已经执行完成的值

以上例Dockerfile文件为例，其中ADD指令在build的过程中就已经将nginx的源码包解压到了指定目录，即便在run的过程中修改`${Web_Server_Package}`nginx源码包的版本，也无法更改已经解压的nginx文件

```shell
Dockerfile  docker build构建镜像             docker run启动容器
   文件    ----------------------> 镜像文件 --------------------> 容器
```

Dockerfile中定义的所有环境变量是容器启动以后可以直接在容器中引用的变量，`docker build`的过程中已经将Dockerfile中的传参做成了只读镜像，启动容器时可以向容器内传参以修改环境变量，但`docker run`传参只是显示环境变量已修改，它并不会修改`docker build`已经产生的结果，`docker build`的过程中已经将环境变量的值做成了只读镜像，`docker run`只是在容器的读写层修改了环境变量，并覆盖了只读镜像的环境变量

<span id="run">**RUN**</span>

用于指定**docker build过程中**运行的程序，可以是任何命令，但使用的命令在基础镜像中必须存在，对比CMD指令，在Dockerfile文件中可以存在多个RUN指令，并逐一运行RUN指令，建议使用换行符一行执行多条命令。语法格式：

```shell
RUN <command> 或
RUN ["<executable>","<param1>","<param2>"]
```

- 第一种格式中，<command>通常是一个shell命令，系统默认会将<command>认为是shell的子命令并以"/bin/sh -c"来运行它，这意味着此进程在容器中PID不为1，不能接收Unix信号，因此，当使用docker stop <container>命令停止容器时，此进程接收不到SIGTERM信号，为了确保该容器能够自动接收UNIX信号，容器会自动执行exec将启动的进程的PID切换为1
- 第二种语法格式的参数是一个json数组，<executable>是要运行的命令，<paramN>作为传递给命令的选项或参数；这种格式的命令不会以"/bin/sh -c"来发起，它直接由内核创建，因此常见的shell操作如变量替换以及通配符替换将会不支持。如果要运行的命令需要依赖shell特性，也可以通过手动启动shell进程的方式实现：RUN ["/bin/bash", "-c", "<executable>", "<param1>"]

在Linux系统下运行一个服务或程序时，其父进程一定会是shell，所有进程在被终止时一定会将其下的所有子进程一并终止，shell也是如此。而想要不被终止则只能绕过shell，直接通过内核启动程序（nohub command），直接通过内核启动的程序不会自动释放内存，也不会随着shell的终止而自动终止，但通过内核直接启动程序却不能使用通配符、管道、重定向等功能，因为这些功能是由shell提供的特性

回到容器，docker的核心理念是一个容器只运行一个进程。而这个进程是直接通过内核启动还是通过shell托管就非常关键，通过容器启动一个进程时，这个进程在整个namespaces中进程号为1，也就是说这个进程由内核启动，所以在命令上就不能使用shell特性，Dockerfile中的RUN命令中的两种语法格式就代表两种启动程序的方式 

启动容器时，如果默认是以shell的方式启动进程，为了确保该容器能够自动接收UNIX信号，容器会自动执行exec将启动的进程的PID切换为1。以`CMD /bin/httpd -f -h /data/web/html/`为例，制作的镜像文件默认会以shell子进程的方式启动httpd服务，使用exec进入已启动的容器后再使用ps命令查看进程时，会发现httpd进程的PID为1，但通过inspect查看镜像或容器详细信息，都能够确认httpd是shell的子进程。确保该容器能够自动接收UNIX信号，是为了管理员在宿主机命令行执行docker stop、docker kill等命令时，容器能够接收到信号并作出相应动作

<span id="cmd">**CMD**</span>

用于指定**docker run过程中**运行的命令，即启动容器时默认要运行的程序，容器默认只运行一个程序，基于这个特性，即便CMD指令可以在Dockerfile文件中出现多次，但只有最后一个生效。CMD指定的命令可以被docker run的命令行指定的命令覆盖。语法格式：

```shell
CMD <command> 或
CMD ["<executable>","<param1>","<param2>"] 或	#前两种语法格式要求与CMD相同
CMD ["<param1>","<param2>"]		#此语法则用于为ENTRYPOINT指令提供默认参数
```

使用CMD的第二种语法格式时需要注意参数的次序位置不能混乱，因为启动容器时读取执行参数也是按照位置次序依次读取执行的

CMD指令与RUN指令对比，两者运行的时间阶段不同，RUN指令运行于构建镜像文件的过程，CMD指令运行于启动容器的过程

<span id="entrypoint">**ENTRYPOINT**</span>

类似CMD指令的功能，为容器指定默认运行程序，但ENTRYPOINT启动的程序不会被docker run命令行指定的参数覆盖，但这些**命令行参数**会覆盖CMD指令的内容，并会附加到ENTRYPOINT指定的应用程序后作为其**参数**使用。不过docker run命令的--entrypoint选项可覆盖ENTRYPOINT指令指定的程序。 语法格式：

```shell
ENTRYPOINT <command>
ENTRYPOINT ["<executable>","<param1>","<param2>"]
```

<span id="user">**USER**</span>

用于指定运行镜像时或运行Dockerfile中任何RUN、CMD或ENTRYPOINT指令指定的程序时的用户名或UID。默认容器运行身份为root。语法格式：

```shell
USER <UID>|<UserName>	#UID必须是/etc/passwd中已存在的用户
```

#### HEALTHCHECK

* docker判断容器正常与否并不是判断容器内进程是否正常提供服务，而是仅判断容器进程是否运行。这种判定机制并不能真正的判断容器是否正常，此时需要用一些工具去测试服务是否正常，比如curl、wget。HEALTHCHECK指令用于指定一条CMD命令，这条命令用于检查主进程服务状态
* 语法格式
```shell
HEALTHCHECK [OPTIONS] CMD command	#CMD是关键词，不可少
HEALTHCHECK NONE	#不使用健康检查，会关闭默认的健康检查
```
HEALTHCHECK属于周期性任务，所以会有OPTIONS：
```shell
--interval=DURATION(default:30s)	#检测间隔时间
--timeout=DURATION(default:30s)		#服务未响应超时时间
--start-period=DURATION(default:0s)	#第一次检测延时时间
--retries=N(default:3)				#检测次数
```

#### SHELL
* 用于指定运行程序使用的shell，Linux默认shell为["/bin/sh","-c"]，Windows默认shell为["cmd","\S","\C"]

#### STOPSIGNAL
* 容器内PID为1的进程可以接收Linux的信号，默认信号是15，也就是终止程序。此信号可修改，比如修改为9，修改后docker stop容器进程收到的信号就是强制终止
* 语法格式
```shell
STOPSIGNAL signal	#修改容器主进程可以接收的信号
```

#### ARG
* 定义只在build的过程中使用的变量，且能在执行docker build命令时，通过选项--build-arg <varname>=<value>向biuld过程中传参
* 语法格式
```shell
ARG <name>[=<default value>]	#此处可以不定义默认值，docker build时传参即可
```

#### ONBUILD
* 在Dockerfile中定义一个触发器，自己在docker build基于此Dockerfile制作的镜像时没有问题，但制作的镜像如果被他人FROM引用做基础镜像时，在他人docker build的过程中会执行此触发器
* 语法格式
```shell
ONBUILD <INSTRUCTION>	#INSTRUCTION可以是RUN、CMD或ADD等
ONBUILD ADD http://mirrors.aliyun.com/repo/Centos-7.repo
```
* 几乎任何指令都可以成为触发器指令，但ONBUILD不能自我嵌套，且不会出发FROM和MAINTAINER指令，使用包含ONBUILD指令的Dockerfile构建的镜像应该使用特殊的标签进行声明。在ONBUILD指令中使用ADD或COPY指令时应注意，新构建过程的上下文在缺少指定的源文件时会失败

```diff
- 构建Dockerfile文件时应尽量减少文件行数，因为构建镜像时每一行就是一层镜像，过多的行数会导致镜像臃肿
```

### 向容器中传参
示例：Dockerfile文件内容
```shell
FROM nginx:1.14-alpine
LABEL maintainer="hebor hebo1248@163.com"
ENV NGX_DOC_ROOT="/data/web/html/"			#指定nginx的网页文件目录
EXPOSE 80									#开放80端口
ADD index.html ${NGX_DOC_ROOT}
ADD entrypoint.sh /bin/						#添加nginx的配置文件脚本
CMD ["/usr/sbin/nginx","-g","daemon off;"]	#设置nginx前台运行
ENTRYPOINT ["/bin/entrypoint.sh"]			#执行nginx配置文件脚本
```
示例：entrypoint.sh脚本内容
```shell
#!/bin/sh
cat > /etc/nginx/conf.d/www.conf << EOF
server {
	server_name $HOSTNAME;
	listen ${IP:-0.0.0.0}:${PORT:-80};		#指定默认IP和端口
	root ${NGX_DOC_ROOT:-/usr/share/nginx/html};
}
EOF				#以上都是nginx配置

exec "$@"		#将CMD的参数作为命令，用exec运行，如果docker run的过程中指定了其他参数，则exec会运行docker run命令行参数
```
