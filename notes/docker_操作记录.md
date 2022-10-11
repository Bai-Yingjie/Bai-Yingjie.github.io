- [docker加proxy](#docker加proxy)
- [docker限制CPU和内存](#docker限制cpu和内存)
- [使用nsenter进入容器](#使用nsenter进入容器)
- [清除不用的image](#清除不用的image)
- [清除不用的container](#清除不用的container)
- [重载entrypoint启动](#重载entrypoint启动)
- [解决连接`/var/run/docker.sock`权限问题](#解决连接varrundockersock权限问题)
- [docker用root登陆](#docker用root登陆)
- [docker国内代理](#docker国内代理)
  - [20210123更新](#20210123更新)
    - [1、配置镜像地址](#1配置镜像地址)
    - [2、国内镜像地址](#2国内镜像地址)
  - [2020 7月](#2020-7月)
- [docker ubuntu for golang](#docker-ubuntu-for-golang)
- [gentoo docker](#gentoo-docker)
- [清理空间](#清理空间)
- [docker保存镜像](#docker保存镜像)
- [history](#history)
- [进入shell](#进入shell)
- [获取每个container的ip](#获取每个container的ip)
- [docker run](#docker-run)
- [docker commit](#docker-commit)
- [容易混淆的概念入门讲解](#容易混淆的概念入门讲解)
- [docker tag可以push到私有库?](#docker-tag可以push到私有库)
- [在docker里面起systemd?](#在docker里面起systemd)
- [Dockerfile: ENTRYPOINT vs CMD](#dockerfile-entrypoint-vs-cmd)
- [Repository和Registry](#repository和registry)
- [用新的command启动一个container](#用新的command启动一个container)
- [image和container的区别?](#image和container的区别)
  - [What's an Image?](#whats-an-image)

# docker加proxy
在docker run命令行加:
```sh
--env http_proxy="http://10.158.100.6:8080/"
```

# docker限制CPU和内存
限制只有2个CPU, 2G内存
```sh
docker run --cpus=2 -m 2g --rm --runtime=runsc -it --name=test centos:7 bash
```
参考: https://docs.docker.com/config/containers/resource_constraints/

举例:
If you have 1 CPU, each of the following commands guarantees the container at most 50% of the CPU every second.
```sh
$ docker run -it --cpus=".5" ubuntu /bin/bash
```
Which is the equivalent to manually specifying `--cpu-period` and `--cpu-quota`;
```sh
$ docker run -it --cpu-period=100000 --cpu-quota=50000 ubuntu /bin/bash
```

# 使用nsenter进入容器
1. `docker ps`找到container ID, 比如是`908592cfba96`
2. 找到这个容器的pid `docker inspect -f{{.State.Pid}} 908592cfba96`, 比如得到10394
3. 进入容器(需要root): `nsenter -m -t 10394 bash`
4. 容器里看到的和`docker exec -it 908592cfba96 bash`一样
注: 这个10394进程一般就是对应containerd-shim的子进程.
> nsenter - run program with namespaces of other processes
> 原理大概就是clone子进程的时候加各种new标记, 比如CLONE_NEWNS, CLONE_NEWIPC, CLONE_NEWPID等等.
默认是进入一个新的name space, 并执行命令. 后面执行的命令对host的空间没影响.
用`-t`选项可以理解为attach到target pid的name space.

# 清除不用的image
The `docker image prune` command allows you to clean up unused images. By default, `docker image prune` only cleans up _dangling_ images. A dangling image is one that is not tagged and is not referenced by any container. To remove dangling images:
```sh
docker image prune
```

# 清除不用的container
When you stop a container, it is not automatically removed unless you started it with the `--rm` flag. To see all containers on the Docker host, including stopped containers, use `docker ps -a`. You may be surprised how many containers exist, especially on a development system! A stopped container’s writable layers still take up disk space. To clean this up, you can use the `docker container prune` command.
```
$ docker container prune
```

# 重载entrypoint启动
有的docker image起不来, 是因为默认的entrypoint启动失败. 此时重载entrypoint到bash一般可启动
```sh
docker run -itd --user $(id -u):$(id -g) -v /etc/passwd:/etc/passwd:ro -v /etc/group:/etc/group:ro -v /home/$(whoami):/home/$(whoami) -v /repo/$(whoami):/repo/$(whoami) -w /repo/$(whoami) --entrypoint=/bin/bash docker-registry-remote.artifactory-blr1.int.net.nokia.com/codercom/code-server
```
对于entrypoint为空的docker
```sh
docker run --rm -itd --user $(id -u):$(id -g) -v /etc/passwd:/etc/passwd:ro -v /etc/group:/etc/group:ro -v /home/$(whoami):/home/$(whoami) -v /repo/$(whoami):/repo/$(whoami) -w /repo/$(whoami) godevsig/godev-tool /bin/bash
fef7fdacb79a78c2e8467e1ff6cfdd38ac0e7072eebfa30df33b8f2805faa020
docker attach fef7fdacb79a78c2e8467e1ff6cfdd38ac0e7072eebfa30df33b8f2805faa020
```

# 解决连接`/var/run/docker.sock`权限问题
将用户加入到docker 组
```sh
sudo usermod -a -G docker $USER
```
# docker用root登陆
一般的基础镜像比如ubuntu, 不知道root密码, 但可以这样登陆
```
docker exec -it -u root cf13ed39f0d9 bash
```

# docker国内代理
## 20210123更新
似乎我后来改用了aliyun的镜像...
```sh
cat /etc/docker/deamon.json
{
    "registry-mirrors": [ "https://ryh6l7pc.mirror.aliyuncs.com" ]
}
```

### 1、配置镜像地址
Docker客户端版本大于 1.10.0 的用户
可以通过修改daemon配置文件`/etc/docker/daemon.json`来使用加速器
```json
{
"registry-mirrors": [ "https://docker.mirrors.ustc.edu.cn" ]
}
```
重启docker和deamon
```sh
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 2、国内镜像地址
也可以免费使用阿里云的镜像。  
登陆阿里云，然后访问：https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors
按照官方提供的文档进行操作即可。
镜像地址：
1) 阿里云 docker hub mirror
https://registry.cn-hangzhou.aliyuncs.com
2) 腾讯云 docker hub mirror
https://mirror.ccs.tencentyun.com
3) 华为云
https://05f073ad3c0010ea0f4bc00b7105ec20.mirror.swr.myhuaweicloud.com
4) docker中国
https://registry.docker-cn.com
5) 网易
http://hub-mirror.c.163.com
6) daocloud
http://f1361db2.m.daocloud.io

## 2020 7月
我用微软的Azure的代理(dockerhub.azk8s.cn)不错:
详见: https://github.com/Azure/container-service-for-azure-china/blob/master/aks/README.md#22-container-registry-proxy
```sh
docker pull dockerhub.azk8s.cn/yingjieb/godev-vscode:latest
```
有人总结的国内列表:https://www.jianshu.com/p/5a911f20d93e
```sh
镜像加速器    镜像加速器地址    专属加速器？    其它加速？
Docker 中国官方镜像    https://registry.docker-cn.com        Docker Hub
DaoCloud 镜像站    http://f1361db2.m.daocloud.io    可登录，系统分配    Docker Hub
Azure 中国镜像    https://dockerhub.azk8s.cn        Docker Hub、GCR、Quay
科大镜像站    https://docker.mirrors.ustc.edu.cn        Docker Hub、GCR、Quay
阿里云    https://<your_code>.mirror.aliyuncs.com    需登录，系统分配    Docker Hub
七牛云    https://reg-mirror.qiniu.com        Docker Hub、GCR、Quay
网易云    https://hub-mirror.c.163.com        Docker Hub
腾讯云    https://mirror.ccs.tencentyun.com        Docker Hub
```

# docker ubuntu for golang
godev 
```sh
docker run --rm -itd --privileged -h yubuntu --name yubuntu -v /repo/yingjieb:/repo/$(whoami) -w /repo/$(whoami) yingjieb/godev/ubuntu
docker attach yubuntu
docker run --rm -itd -h yubuntu -v /repo/yingjieb:/repo/$(whoami) -w /repo/$(whoami) yingjieb/ubuntu-godev
docker run --privileged -itd -h yubuntu --name yubuntu -v /repo/yingjieb:/repo/$(whoami) -w /repo/$(whoami) yingjieb/ubuntu-godev
#登陆docker后
usermod -u 3030425 godev
groupmod -g 1155 godev
su godev
```

# gentoo docker
gentoo的docker image也是基于stage3的包, 不同的是, 它修改了/etc/rc.conf

详见
https://github.com/gentoo/gentoo-docker-images/blob/master/stage3.Dockerfile
```sh
# This is the subsystem type.
# It is used to match against keywords set by the keyword call in the
# depend function of service scripts.
#
# It should be set to the value representing the environment this file is
# PRESENTLY in, not the virtualization the environment is capable of.
# If it is commented out, automatic detection will be used.
#
# The list below shows all possible settings as well as the host
# operating systems where they can be used and autodetected.
#
# ""               - nothing special
# "docker"         - Docker container manager (Linux)
# "jail"           - Jail (DragonflyBSD or FreeBSD)
# "lxc"            - Linux Containers
# "openvz"         - Linux OpenVZ
# "prefix"         - Prefix
# "rkt"            - CoreOS container management system (Linux)
# "subhurd"        - Hurd subhurds (to be checked)
# "systemd-nspawn" - Container created by systemd-nspawn (Linux)
# "uml"            - Usermode Linux
# "vserver"        - Linux vserver
# "xen0"           - Xen0 Domain (Linux and NetBSD)
# "xenU"           - XenU Domain (Linux and NetBSD)
rc_sys="docker"
```

# 清理空间
```sh
sudo systemctl stop docker
#删除前先备份
rm -rf /var/lib/docker
```

# docker保存镜像
* 对image
```sh
docker save 66389e4e65c5 -o centos-aarch64:7.4.tar
docker load -i arm64-gentoo.1.0.tar
docker tag f43a088c78f5 arm64-gentoo:1.0
```

* 对container
```sh
docker export 239fc2b54d18 -o arm64-gentoo.2.0.tar
docker import -c 'CMD ["/bin/bash"]' arm64-gentoo.1.2.0.tar
```

* 二者区别是export只导出rootfs, 丢失元数据层

# history
```sh
# 执行docker instance里面的一个命令
sudo docker exec 080443f42a7d ip -s addr
docker pull docker-registry.qualcomm.com/yingjieb/centos-aarch64:7.4
docker login -u="yingjieb" -p="ReU6z0UtlODAt210c9fn+gSMhszgMT/9Bkr4D6fMAsayPz6wmEKB0tFNN7xaIX2N" docker-registry.qualcomm.com
docker commit bbacde9f6753 docker-registry.qualcomm.com/yingjieb/centos-aarch64:7.4
docker push docker-registry.qualcomm.com/yingjieb/centos-aarch64:7.4
docker run --privileged -ti -u qdt -h dcent docker-registry.qualcomm.com/yingjieb/centos-aarch64:7.4
docker run --privileged -it -u qdt -h dcent --name=ceph-mon1 docker-registry.qualcomm.com/yingjieb/centos-aarch64:7.4
docker run --privileged -dt -u qdt -h dcent --name=ceph-mon2 docker-registry.qualcomm.com/yingjieb/centos-aarch64:7.4
# 上面两个命令只有-it和-dt不一样, -t是说要创建个tty, 而i是要进入交互模式, d是detach模式(后台运行); 需要注意着都和command有关, 这个image默认的command是/bin/bash, 是不退出的(不加选项t会退出, 因为bash必须运行在tty环境下?), 所以it还是dt都会一直运行;
# 这里itd可以一起使用, 这样既能在后台继续运行, 又保留的shell交互入口
docker run --privileged -itd -u qdt -h dcent --name=ceph-mon3 docker-registry.qualcomm.com/yingjieb/centos-aarch64:7.4
docker run -itd --privileged -u qdt -h dgentoo --name=dgentoo arm64-gentoo:2.0
docker attach ceph-mon3; 可以进入shell, 退出时ctrl+p ctrl+q
docker start -ai ceph-mon1 启动一个创建好的实例, attach并进入交互模式
docker cp linux-4.14.tar.xz agitated_wright:/root
```

# 进入shell
```sh
docker exec -it ceph-mon2 /bin/bash
```

# 获取每个container的ip
```sh
docker inspect -f '{{.Name}} - {{.NetworkSettings.IPAddress}}' $(docker ps -aq)
```

# docker run
docker run有限制资源使用的N多选项
比较有用的选项
```sh
--cpuset-cpus=""
          CPUs in which to allow execution (0-3, 0,1)
--cpuset-mems=""
          Memory nodes (MEMs) in which to allow execution (0-3, 0,1). Only effective on NUMA systems.
--device=[]
          Add a host device to the container (e.g. --device=/dev/sdc:/dev/xvdc:rwm)
-h, --hostname=""
          Container host name
--expose=[]
          Expose a port, or a range of ports (e.g. --expose=3300-3310) informs Docker that the container listens on the  specified  network  ports  at
       runtime. Docker uses this information to interconnect containers using links and to set up port redirection on the host system.
-e, --env=[]
          Set environment variables
       This  option  allows you to specify arbitrary environment variables that are available for the process that will be launched inside of the con‐
       tainer.
--entrypoint=""
          Overwrite the default ENTRYPOINT of the image
--ip=""
          Sets the container's interface IPv4 address (e.g. 172.23.0.9)
       It can only be used in conjunction with --net for user-defined networks
-m, --memory=""
          Memory limit (format: <number>[<unit>], where unit = b, k, m or g)
--name=""
          Assign a name to the container
--net="bridge"
          Set the Network mode for the container
                                      'bridge': create a network stack on the default Docker bridge
                                      'none': no networking
                                      'container:<name|id>': reuse another container's network stack
                                      'host':  use  the Docker host network stack. Note: the host mode gives the container full access to local system
       services such as D-bus and is therefore considered insecure.
                                      '<network-name>|<network-id>': connect to a user-defined network
-p, --publish=[]
          Publish a container's port, or range of ports, to the host.
       Format: ip:hostPort:containerPort | ip::containerPort | hostPort:containerPort | containerPort Both hostPort and containerPort can be specified
       as a range of ports.  When specifying ranges for both, the number of container ports in the range must match the number of host  ports  in  the
       range.   (e.g.,  docker run -p 1234-1236:1222-1224 --name thisWorks -t busybox but not docker run -p 1230-1236:1230-1240 --name RangeContainer‐
       PortsBiggerThanRangeHostPorts -t busybox) With ip: docker run -p 127.0.0.1:$HOSTPORT:$CONTAINERPORT --name CONTAINER -t  someimage  Use  docker
       port to see the actual mapping: docker port CONTAINER $CONTAINERPORT
--privileged=true|false
          Give extended privileges to this container. The default is false.
       By default, Docker containers are “unprivileged” (=false) and cannot, for example, run a Docker daemon inside the  Docker  container.  This  is
       because by default a container is not allowed to access any devices. A “privileged” container is given access to all devices.
       When  the  operator executes docker run --privileged, Docker will enable access to all devices on the host as well as set some configuration in
       AppArmor to allow the container nearly all the same access to the host as processes running outside of a container on the host.
-u, --user=""
          Sets the username or UID used and optionally the groupname or GID for the specified command.
       The followings examples are all valid:
          --user [user | user:group | uid | uid:gid | user:gid | uid:group ]
       Without this argument the command will be run as root in the container.
-v|--volume[=[[HOST-DIR:]CONTAINER-DIR[:OPTIONS]]]
          Create a bind mount. If you specify, -v /HOST-DIR:/CONTAINER-DIR, Docker
          bind mounts /HOST-DIR in the host to /CONTAINER-DIR in the Docker
          container. If 'HOST-DIR' is omitted,  Docker automatically creates the new
          volume on the host.  The OPTIONS are a comma delimited list and can be:
       0
              item [rw|ro] item [z|Z] item [[r]shared|[r]slave|[r]private] item [nocopy]
-w, --workdir=""
          Working directory inside the container
       The default working directory for running binaries within a container is the root directory (/). The developer can set a different default with
       the Dockerfile WORKDIR instruction. The operator can override the working directory by using the -w option.
Mapping Ports for External Usage
       The  exposed port of an application can be mapped to a host port using the -p flag. For example, a httpd port 80 can be mapped to the host port
       8080 using the following:
              # docker run -p 8080:80 -d -i -t fedora/httpd
Creating and Mounting a Data Volume Container
       Many applications require the sharing of persistent data across several containers. Docker allows you to create a Data  Volume  Container  that
       other  containers can mount from. For example, create a named container that contains directories /var/volume1 and /tmp/volume2. The image will
       need to contain these directories so a couple of RUN mkdir instructions might be required for you fedora-data image:
              # docker run --name=data -v /var/volume1 -v /tmp/volume2 -i -t fedora-data true
              # docker run --volumes-from=data --name=fedora-container1 -i -t fedora bash
       Multiple --volumes-from parameters will bring together multiple data volumes from multiple containers. And it's possible to mount  the  volumes
       that  came  from  the DATA container in yet another container via the fedora-container1 intermediary container, allowing to abstract the actual
       data source from users of that data:
              # docker run --volumes-from=fedora-container1 --name=fedora-container2 -i -t fedora bash
Mounting External Volumes
       To mount a host directory as a container volume, specify the absolute path to the directory and the absolute path for the  container  directory
       separated by a colon:
              # docker run -v /var/db:/data1 -i -t fedora bash
```

# docker commit
https://docs.docker.com/engine/reference/commandline/commit/

可以用container生成一个image  
举例:
普通的commit, 名字随便起?
```sh
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS              NAMES
c3f279d17e0a        ubuntu:12.04        /bin/bash           7 days ago          Up 25 hours                            desperate_dubinsky
197387f1b436        ubuntu:12.04        /bin/bash           7 days ago          Up 25 hours                            focused_hamilton
$ docker commit c3f279d17e0a  svendowideit/testimage:version3
f5283438590d
$ docker images
REPOSITORY                        TAG                 ID                  CREATED             SIZE
svendowideit/testimage            version3            f5283438590d        16 seconds ago      335.7 MB
```
commit的时候还能顺便更改配置
```sh
$ docker ps
CONTAINER ID       IMAGE               COMMAND             CREATED             STATUS              PORTS              NAMES
c3f279d17e0a        ubuntu:12.04        /bin/bash           7 days ago          Up 25 hours                            desperate_dubinsky
197387f1b436        ubuntu:12.04        /bin/bash           7 days ago          Up 25 hours                            focused_hamilton
$ docker inspect -f "{{ .Config.Env }}" c3f279d17e0a
[HOME=/ PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin]
$ docker commit --change "ENV DEBUG true" c3f279d17e0a  svendowideit/testimage:version3
f5283438590d
$ docker inspect -f "{{ .Config.Env }}" f5283438590d
[HOME=/ PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin DEBUG=true]
```
更改CMD和EXPOSE
```sh
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS              NAMES
c3f279d17e0a        ubuntu:12.04        /bin/bash           7 days ago          Up 25 hours                            desperate_dubinsky
197387f1b436        ubuntu:12.04        /bin/bash           7 days ago          Up 25 hours                            focused_hamilton
$ docker commit --change='CMD ["apachectl", "-DFOREGROUND"]' -c "EXPOSE 80" c3f279d17e0a  svendowideit/testimage:version4
f5283438590d
$ docker run -d svendowideit/testimage:version4
89373736e2e7f00bc149bd783073ac43d0507da250e999f3f1036e0db60817c0
$ docker ps
CONTAINER ID        IMAGE               COMMAND                 CREATED             STATUS              PORTS              NAMES
89373736e2e7        testimage:version4  "apachectl -DFOREGROU"  3 seconds ago       Up 2 seconds        80/tcp             distracted_fermat
c3f279d17e0a        ubuntu:12.04        /bin/bash               7 days ago          Up 25 hours                            desperate_dubinsky
197387f1b436        ubuntu:12.04        /bin/bash               7 days ago          Up 25 hours                            focused_hamilton
```

# 容易混淆的概念入门讲解
http://blog.thoward37.me/articles/where-are-docker-images-stored/

# docker tag可以push到私有库?
Tagging an image for a private repository
To push an image to a private registry and not the central Docker registry you must tag it with the registry hostname and port (if needed).
```sh
docker tag 0e5574283393 myregistryhost:5000/fedora/httpd:version1.0
```

# 在docker里面起systemd?

> A package with the systemd initialization system is included in the official Red Hat Enterprise Linux base images. This means that applications created to be managed with systemd can be started and managed inside a container. A container running systemd will:
NOTE
Previously, a modified version of the systemd initialization system called systemd-container was included in the Red Hat Enterprise Linux versions 7.2 base images. Now, the systemd package is the same across systems.
Start the /sbin/init process (the systemd service) to run as PID 1 within the container.
Start all systemd services that are installed and enabled within the container, in order of dependencies.
Allow systemd to restart services or kill zombie processes for services started within the container.
The general steps for building a container that is ready to be used as a systemd services is:
Install the package containing the systemd-enabled service inside the container. This can include dozens of services that come with RHEL, such as Apache Web Server (httpd), FTP server (vsftpd), Proxy server (squid), and many others. For this example, we simply install an Apache (httpd) Web server.
Use the systemctl command to enable the service inside the container.
Add data for the service to use in the container (in this example, we add a Web server test page). For a real deployment, you would probably connect to outside storage.
Expose any ports needed to access the service.
Set /sbin/init as the default process to start when the container runs
In this example, we build a container by creating a Dockerfile that installs and configures a Web server (httpd) to start automatically by the systemd service (/sbin/init) when the container is run on a host system.
1 Create Dockerfile: In a separate directory, create a file named Dockerfile with the following contents:

```sh
FROM rhel7
RUN yum -y install httpd; yum clean all; systemctl enable httpd;
RUN echo "Successful Web Server Test" > /var/www/html/index.html
RUN mkdir /etc/systemd/system/httpd.service.d/; echo -e '[Service]\nRestart=always' > /etc/systemd/system/httpd.service.d/httpd.conf
EXPOSE 80
CMD [ "/sbin/init" ]
```
> The Dockerfile installs the httpd package, enables the httpd service to start at boot time (i.e. when the container starts), creates a test file (index.html), exposes the Web server to the host (port 80), and starts the systemd init service (/sbin/init) when the container starts.
2 Build the container: From the directory containing the Dockerfile, type the following:
`# docker build -t mysysd .`
3 Run the container: Once the container is built and named mysysd, type the following to run the container:
`# docker run -d --name=mysysd_run -p 80:80 mysysd`
From this command, the mysysd image runs as the mysysd_run container as a daemon process, with port 80 from the container exposed to port 80 on the host system.
4 Check that the container is running: To make sure that the container is running and that the service is working, type the following commands:
```sh
# docker ps | grep mysysd_run
de7bb15fc4d1   mysysd   "/sbin/init"   3 minutes ago   Up 2 minutes   0.0.0.0:80->80/tcp   mysysd_run
# curl localhost/index.html
Successful Web Server Test
```
> At this point, you have a container that starts up a Web server as a systemd service inside the container. Install and run any services you like in this same way by modifying the Dockerfile and configuring data and opening ports as appropriate.

# Dockerfile: ENTRYPOINT vs CMD
https://www.ctl.io/developers/blog/post/dockerfile-entrypoint-vs-cmd/

简单来说, CMD和ENTRYPOINT都可以指定进docker以后, 默认的执行程序, 两者都可以被重载, 重载CMD更容易. 重载ENTRYPOINT需要加额外参数

# Repository和Registry
* Repository：本身是一个仓库，这个仓库里面可以放具体的镜像，是指具体的某个镜像的仓库，比如Tomcat下面有很多个版本的镜像，它们共同组成了Tomcat的Repository。
* Registry：镜像的仓库，比如官方的是Docker Hub，它是开源的，也可以自己部署一个，Registry上有很多的Repository，Redis、Tomcat、MySQL等等Repository组成了Registry。

# 用新的command启动一个container
Find your stopped container id
`docker ps -a`

Commit the stopped container:
This command saves modified container state into a new image user/test_image
`docker commit $CONTAINER_ID user/test_image`

Start/run with a different entry point:
`docker run -ti --entrypoint=sh user/test_image`

# image和container的区别?
container是image的一个实例

## What's an Image?
An image is an inert, immutable, file that's essentially a snapshot of a container. Images are created with the build command, and they'll produce a container when started with run. Images are stored in a Docker registry such as registry.hub.docker.com. Because they can become quite large, images are designed to be composed of layers of other images, allowing a miminal amount of data to be sent when transferring images over the network.
Local images can be listed by running docker images:

```
REPOSITORY                TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
ubuntu                    13.10               5e019ab7bf6d        2 months ago        180 MB
ubuntu                    14.04               99ec81b80c55        2 months ago        266 MB
ubuntu                    latest              99ec81b80c55        2 months ago        266 MB
ubuntu                    trusty              99ec81b80c55        2 months ago        266 MB
<none>                    <none>              4ab0d9120985        3 months ago        486.5 MB
```
Some things to note:
* IMAGE ID is the first 12 characters of the true identifier for an image. You can create many tags of a given image, but their IDs will all be the same (as above).
* VIRTUAL SIZE is virtual because its adding up the sizes of all the distinct underlying layers. This means that the sum of all the values in that column is probably much larger than the disk space used by all of those images.
* The value in the REPOSITORY column comes from the -t flag of the docker build command, or from docker tag-ing an existing image. You're free to tag images using a nomenclature that makes sense to you, but know that docker will use the tag as the registry location in a docker push or docker pull.
* The full form of a tag is [REGISTRYHOST/][USERNAME/]NAME[:TAG]. For ubuntu above, REGISTRYHOST is inferred to be registry.hub.docker.com. So if you plan on storing your image called my-application in a registry at docker.example.com, you should tag that image docker.example.com/my-application.
* The TAG column is just the [:TAG] part of the full tag. This is unfortunate terminology.
* The latest tag is not magical, it's simply the default tag when you don't specify a tag.
* You can have untagged images only identifiable by their IMAGE IDs. These will get the <none> TAG and REPOSITORY. It's easy to forget about them.
More info on images is available from the Docker docs and glossary.
What's a container?
To use a programming metaphor, if an image is a class, then a container is an instance of a class—a runtime object. Containers are hopefully why you're using Docker; they're lightweight and portable encapsulations of an environment in which to run applications.
View local running containers with docker ps:
```
CONTAINER ID        IMAGE                               COMMAND                CREATED             STATUS              PORTS                    NAMES
f2ff1af05450        samalba/docker-registry:latest      /bin/sh -c 'exec doc   4 months ago        Up 12 weeks         0.0.0.0:5000->5000/tcp   docker-registry
```
Here I'm running a dockerized version of the docker registry, so that I have a private place to store my images. Again, some things to note:
Like IMAGE ID, CONTAINER ID is the true identifier for the container. It has the same form, but it identifies a different kind of object.
docker ps only outputs running containers. You can view all containers (running or stopped) with docker ps -a.
NAMES can be used to identify a started container via the --name flag.
