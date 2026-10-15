#              DOCKER实践篇(1-3)

## 一、Docker部署

### 1.1清除podman包

先建立一个docker虚拟机，再用远程连接软件连上去

```
[root@docker ~]# rpm -qa   #查询本机已经安装的所有 rpm 软件包
[root@docker ~]# rpm -qa | grep podman #检测是否有podman包
```

![image-20261008183606898](C:\Users\33564\AppData\Roaming\Typora\typora-user-images\image-20261008183606898.png)

如果有podman包，用remove删除此包

```
[root@docker ~]# dnf remove  podman-4.6.1-5.el9.x86_64 -y
[root@docker ~]# rpm -qa | grep podman
```

![image-20261008183819147](C:\Users\33564\AppData\Roaming\Typora\typora-user-images\image-20261008183819147.png)

这个状态就是无podman包的情况

### 1.2部署docker

#### 1.2.1配置软件仓库

```bash
[root@docker ~]# cd /etc/yum.repos.d
[root@docker yum.repos.d]# vim docker.repo
[docker]
name=docker-ce
baseurl=https://mirrors.aliyun.com/docker-ce/linux/rhel/9.3/x86_64/stable/
gpgcheck=0
[root@docker yum.repos.d]# dnf list docker-ce --showduplicates #把源里面的所有docker列出来
```

#### 1.2.2安装docker-ce并启动服务

```bash
[root@docker yum.repos.d]# dnf install docker-ce --downloadonly --downloaddir /mnt/      #不安装直接下载到/mnt/
[root@docker yum.repos.d]# cd /mnt
[root@docker mnt]# ls              #下载的包
containerd.io-2.3.6-1.el9.x86_64.rpm
docker-buildx-plugin-0.37.1-1.el9.x86_64.rpm
docker-ce-29.8.2-1.el9.x86_64.rpm
docker-ce-cli-29.8.2-1.el9.x86_64.rpm
docker-ce-rootless-extras-29.8.2-1.el9.x86_64.rpm
docker-compose-plugin-5.6.0-1.el9.x86_64.rpm
hgfs
[root@docker mnt]# tar zcf docker-ce-29.8.2.tar.gz *.rpm    #做了一个压缩包，把所有的rpm打包了
[root@docker mnt]# dnf install *.rpm -y  #安装
```

其他主机如果需要使用将docker-ce-29.8.2.tar.gz包拷过去用tar zxf docker-ce-29.8.2.tar.gz这条命令解包

```bash
[root@docker mnt]# ls
docker-ce-29.8.2.tar.gz  hgfs
[root@docker mnt]# tar zxf docker-ce-29.8.2.tar.gz
```

#### 1.2.3**激活内核网络选项**

```bash
[root@docker mnt]# echo br_netfilter>/etc/modules-load.d/docker_mod.conf     
#开机时自动加载docker_mod.conf文件里面的模块，开机自动生效
[root@docker mnt]# modprobe -a br_netfilter      #添加模块，当前生效
[root@docker mnt]# lsmod  | grep br_netfilter
br_netfilter           36864  0
bridge                405504  1 br_netfilter
[root@docker mnt]# vim /etc/sysctl.conf 
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
[root@docker mnt]# sysctl -p      #启动生效
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
[root@docker mnt]# vim /lib/systemd/system/docker.service   #编写docker的启动文件
```

![image-20261008194227016](C:\Users\33564\AppData\Roaming\Typora\typora-user-images\image-20261008194227016.png)

在这个模块里的E开头的那句后面加--iptables=true

```bash
[root@docker mnt]# systemctl enable --now docker        #启动
[root@docker mnt]# docker --version
Docker version 29.8.2, build 7fc2dff
```

## 二、**Docker**容器

### 2.1拉取nginx

#### 见3.1.1 可以不用配置加速器使用科学上网软件 输入docker pull nginx 

```bash
[root@docker ~]# vim /etc/docker/daemon.json
{
  "registry-mirrors":["https://docker.m.daocloud.io"]
}
[root@docker ~]# systemctl restart docker.service
[root@docker ~]# docker pull nginx
Using default tag: latest
latest: Pulling from library/nginx
f7e1db57b58b: Pull complete
ecc510c1e359: Pull complete
04bdbc7ea6b5: Pull complete
807ab930f3a0: Pull complete
1efc53192927: Pull complete
947d22046304: Pull complete
8020170dc65f: Pull complete
a1b85ab28c90: Download complete
c329293b1d90: Download complete
Digest: sha256:f9ea18bfa4fad859e1ed38259d711da7ccad2c3516e875cec3351a57c859f571
Status: Downloaded newer image for nginx:latest
docker.io/library/nginx:latest
[root@docker ~]# docker images
 i Info →   U  In Use
IMAGE          ID             DISK USAGE   CONTENT SIZE   EXTRA
nginx:latest   f9ea18bfa4fa        240MB         66.7MB
[root@docker ~]# docker run -d --name webserver -p 80:80 nginx:latest  #后台启动一个名字叫 webserver 的 nginx 容器
6325a07d2527e41e789609d2033736e1ec97c684e6e49eae4128f63f5674f9b0
```

**nginx拉取成功，访问正确**

![image-20261008210029521](C:\Users\33564\AppData\Roaming\Typora\typora-user-images\image-20261008210029521.png)



### 2.2运行、停止容器

#### （1）运行容器

```bash
[root@docker ~]# docker exec -it webserver /bin/bash    #进入正在运行的webserver 的 nginx 容器内部
root@6325a07d2527:/#
root@6325a07d2527:/# ls
bin   docker-entrypoint.d   home   media  proc  sbin  tmp
boot  docker-entrypoint.sh  lib    mnt    root  srv   usr
dev   etc                   lib64  opt    run   sys   var
root@6325a07d2527:/# cd /usr/share/nginx/
root@6325a07d2527:/usr/share/nginx# ls
html
root@6325a07d2527:/usr/share/nginx# cd html/
root@6325a07d2527:/usr/share/nginx/html# ls
50x.html  index.html
root@6325a07d2527:/usr/share/nginx/html# echo timinglee > index.html
```

![image-20261008211217392](C:\Users\33564\AppData\Roaming\Typora\typora-user-images\image-20261008211217392.png)

#### （2）**停止容器**

```bash
[root@docker ~]# docker stop webserver
webserver
[root@docker ~]# docker rm webserver
webserver
```

## 三、Docker的基本操作

### 3.1Docker镜像管理

#### 3.1.1搜索镜像

原理：`docker pull` 走镜像加速器所以正常；`docker search` 不支持加速器、只能直连官方源，需让 dockerd 走代理。

#### 使用科学上网软件

> ### 1. Clash Verge 开启局域网监听
>
> - 设置 → 系统设置 → **局域网连接（Allow LAN）** → 打开
>
> - 端口设置确认：混合端口 **7897**
>
> - ⚠️ 注意：开关切换后可能不立即生效，需**完全退出 Clash Verge（托盘右键退出）再重新打开**

> ### 2.用win+R打开cmd
>
> ```bash
> 验证监听是否对外
> netstat -ano | findstr 7897 | findstr LISTENING
> - 必须看到 `0.0.0.0:7897` 才算对外监听
> -只有 `127.0.0.1:7897` = Allow LAN 没生效 → 重启 Clash Verge 再查
> ```

> ### 3. 防火墙放行 7897 端口
>
> ```bash
> 1.以管理员身份打开 PowerShell（Win+X → 终端(管理员)，普通窗口会报 System Error 5 权限不足）：
> New-NetFirewallRule -DisplayName "Clash LAN 7897" -Direction Inbound -LocalPort 7897 -Protocol TCP -Action Allow -Profile Any
> 
> 2. 对本机的实际风险：很低，但不是零(开防火墙可能会有风险)
> 这个规则作用在 192.168.168.x 这个 VMware 虚拟网卡（VMnet8） 上。这是私有网段，特点：
>   你家/公司的物理局域网里其他电脑访问不到它（路由器不会往这个虚拟网段转发）
>   互联网上的任何人更不可能访问到（私有地址不可路由，且你没做端口映射）
>   能连上它的，实际只有你这台机器上跑的虚拟机——也就是你自己
> 3.不想用了，怎么彻底删掉
> Remove-NetFirewallRule -DisplayName "Clash LAN 7897"
>    删掉之后，虚拟机那边的 docker search 又会失效（走不了代理了），但 docker pull 走加速器不受影响，内网实验也照常。所以只在你不打算再用 search 时才删。
> ```

> ### 4. 确认宿主机在 VMnet8 上的 IP（在cmd里面查）
>
> ```
> ipconfig
> ```

> ### 5.虚拟机测
>
> #### (1)先测代理连通性
>
> ```bash
> curl -x http://192.168.168.1:7897 -I https://www.google.com    #192.1681.68.1所有容器的网关
> ```
>
> #### (2)写入 Docker 代理配置
>
> ```bash
> mkdir -p /etc/systemd/system/docker.service.d
> tee /etc/systemd/system/docker.service.d/proxy.conf <<'EOF'
> [Service]
> Environment="HTTP_PROXY=http://192.168.168.1:7897"   
> Environment="HTTPS_PROXY=http://192.168.168.1:7897"
> Environment="NO_PROXY=localhost,127.0.0.1,192.168.168.0/24"
> EOF
> systemctl daemon-reload
> systemctl restart docker
> ```
>
> #### (3)验证
>
> ```
> docker search busybox
> ```

```bash
[root@docker docker]# docker search busybox
NAME                   DESCRIPTION                   STARS     OFFICIAL
busybox                Busybox base image.           3521      [OK]
[root@docker docker]# docker search nginx
NAME               DESCRIPTION               STARS     OFFICIAL nginx               Official          build of Nginx.            21407     [OK]
```

#### 3.1.2拉取镜像

```bash
[root@docker docker]# docker pull busybox
Using default tag: latest
latest: Pulling from library/busybox
37bb94b0940b: Pull complete
46500c34ad0c: Download complete
Digest: sha256:fd7dc98638c8e305f4dc34e979f1c0fdfdcaeb0fbf8fcff77ae834b6da3d7e6e
Status: Downloaded newer image for busybox:latest
docker.io/library/busybox:latest
[root@docker docker]# docker images
                                                                      i Info →   U  In Use
IMAGE            ID             DISK USAGE   CONTENT SIZE   EXTRA
busybox:latest   fd7dc98638c8       6.74MB         2.23MB
nginx:latest     f9ea18bfa4fa        240MB         66.7MB
[root@docker docker]# docker run -d --name webserver -p 80:80 nginx:latest
c171dc175f3b2128800459b534f1ff5f4277aaba20a359cc2575b7561bcc83f9
[root@docker docker]# docker images
                                                                      i Info →   U  In Use
IMAGE            ID             DISK USAGE   CONTENT SIZE   EXTRA
busybox:latest   fd7dc98638c8       6.74MB         2.23MB
nginx:latest     f9ea18bfa4fa        240MB         66.7MB    U          #U代表正在运行中
[root@docker docker]# docker stop webserver
[root@docker docker]# docker rm webserver
[root@docker docker]# docker images             #只有stop和rm docker在docker images 才能停止正在运行的容器
                                                                      i Info →   U  In Use
IMAGE            ID             DISK USAGE   CONTENT SIZE   EXTRA
busybox:latest   fd7dc98638c8       6.74MB         2.23MB
nginx:latest     f9ea18bfa4fa        240MB         66.7MB
[root@docker docker]# docker pull nginx:alpine                 #nginx最小版本
```

#### 3.1.3查看镜像信息

```bash
[root@docker docker]# docker image inspect nginx:latest        #查看镜像详细信息
```

#### 3.1.4导出镜像

```bash
[root@docker docker]# docker save -o images.tar.gz busybox:latest nginx:alpine nginx:latest                   #导出多个镜像
```

#### 3.1.5删除镜像

```bash
[root@docker docker]# docker rmi nginx:alpine
Untagged: nginx:alpine
Deleted: sha256:df221db836e1754089190208cee7eeda94f233197056426eda74a43ab1abeac2
[root@docker docker]# docker rmi nginx:latest
[root@docker docker]# docker rmi busybox:latest
[root@docker docker]# docker images
                                                                      i Info →   U  In Use
IMAGE   ID             DISK USAGE   CONTENT SIZE   EXTRA
```

#### 3.1.6如何保存如何加载

```bash
[root@docker docker]# ls
images.tar.gz
[root@docker docker]# docker load -i images.tar.gz    #加载docker
Loaded image: busybox:latest
Loaded image: nginx:alpine
Loaded image: nginx:latest
[root@docker docker]# docker images
                                                                      i Info →   U  In Use
IMAGE            ID             DISK USAGE   CONTENT SIZE   EXTRA
busybox:latest   fd7dc98638c8       6.74MB         2.23MB
nginx:alpine     df221db836e1         93MB         27.2MB
nginx:latest     f9ea18bfa4fa        240MB         66.7MB
```

### 3.2容器的常用操作

#### 3.2.1启动容器

```bash
[root@docker ~]# docker run -d --name webserver -p 80:80 nginx:latest  #非交互运行，-d代表后台运行，--name代表的是给容器起名字，-p是端口映射，前面的80代表的是操作系统，后面的80代表的是容器的端口
5218d06ca7aeec914fbb3b2550ca5218c655e3a767f5ec9b7fd2439e65e76ec1
[root@docker ~]# docker run -it --name busybox busybox:latest   #交互式运行
/ # ip a
/ # exit                                                        #退出容器
[root@docker ~]# docker ps            
CONTAINER ID   IMAGE          COMMAND                   CREATED         STATUS         PORTS                                 NAMES
5218d06ca7ae   nginx:latest   "/docker-entrypoint.…"   5 minutes ago   Up 5 minutes   0.0.0.0:80->80/tcp, [::]:80->80/tcp   webserver
[root@docker ~]# docker start busybox          #又重新运行起容器
[root@docker ~]# docker attach busybox         #进入容器
/ #
/ # read escape sequence                    #按住ctrl+pq就会显示这句话，退出容器了但还在运行
[root@docker ~]# docker ps -a
CONTAINER ID   IMAGE            COMMAND                   CREATED         STATUS          PORTS                                 NAMES
af7e64b4c884   busybox:latest   "sh"                      6 minutes ago   Up 23 seconds                                         busybox
5218d06ca7ae   nginx:latest     "/docker-entrypoint.…"   9 minutes ago   Up 9 minutes    0.0.0.0:80->80/tcp, [::]:80->80/tcp   webserver
[root@docker ~]# docker exec busybox ifconfig              #在容器中执行命令
[root@docker ~]# docker run -it --name  busybox --rm busybox:latest   #容器退出就被删掉
/ #                                                       #按住ctrl+pq
[root@docker ~]# docker attach busybox    
/ # exit
```

#### 3.2.2**查看容器运行信息**

```bash
[root@docker ~]# docker ps       #查看当前运行容器
[root@docker ~]# docker ps -a    #查看所有容器
[root@docker ~]# docker inspect busybox #查看容器运行的详细信息
```

#### 3.2.3**停止和运行容器**

```bash
[root@docker ~]# docker stop busybox #停止容器
[root@docker ~]# docker kill busybox #杀死容器，可以使用信号
[root@docker ~]# docker start busybox #开启停止的容器
```

#### 3.2.4删除容器

```bash
[root@docker ~]# docker rm centos7 #删除停止的容器
[root@docker ~]# docker rm -f busybox #删除运行的容器
[root@docker ~]# docker container prune -f #删除所有停止的容器
```

#### 3.2.5**容器外复制数据**

##### （1）把容器外的文件复制到容器内部

```bash
[root@docker ~]# docker run -d --name webserver -p 80:80 nginx:latest
[root@docker ~]# vim index.html
hello timinglee
[root@docker ~]# docker cp index.html webserver:/usr/share/nginx/html
Successfully copied 16B (transferred 2.05kB) to webserver:/usr/share/nginx/html
[root@docker ~]# docker exec -it webserver /bin/bash
root@a28074394b0c:/# ls /usr/share/nginx/html
50x.html  index.html
root@a28074394b0c:/# cat /usr/share/nginx/html/index.html
hello timinglee
root@a28074394b0c:/# exit
exit
[root@docker ~]# curl 192.168.168.10
hello timinglee
```

##### （2）把容器内的文件复制到容器外

```bash
[root@docker ~]# rm -rf index.html                  #先删除之前的index.html
[root@docker ~]# docker cp webserver:/usr/share/nginx/html/index.html /mnt
Successfully copied 16B (transferred 2.05kB) to /mnt
[root@docker ~]# ls /mnt
index.html
[root@docker ~]# cat /mnt/index.html
hello timinglee
```

#### 3.2.6提交容器更改

默认在容器中所有操作，在容器被删除后就没有了

##### 提交容器的更改

```bash
[root@docker ~]# docker commit -m "add index.html" webserver nginx_index:latest    #提交
sha256:6627dfa68f3b6bf7d8965c719c0e9e0c7ecba6da2856d8fcda5cd19ef91d6974
[root@docker ~]# docker rm -f webserver             #先删掉之前旧的webserver
webserver
[root@docker ~]# docker run -d --name webserver -p 80:80 nginx_index:latest
0a4fdf1b0104845662bdce0acb0ab536b0a3bf99f1fc5d611f8ca0a60cb840b2
[root@docker ~]# docker exec webserver cat /usr/share/nginx/html/index.html
hello timinglee
[root@docker ~]# curl 192.168.168.10
hello timinglee
```

#### 3.2.7查询容器内部日志

```bash
[root@docker ~]# docker logs webserver
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: IPv6 listen already enabled
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2026/10/09 12:33:53 [notice] 1#1: using the "epoll" event method
2026/10/09 12:33:53 [notice] 1#1: nginx/1.31.6
2026/10/09 12:33:53 [notice] 1#1: built by gcc 14.2.0 (Debian 14.2.0-19)
2026/10/09 12:33:53 [notice] 1#1: OS: Linux 5.14.0-362.8.1.el9_3.x86_64
2026/10/09 12:33:53 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1024:524288
2026/10/09 12:33:53 [notice] 1#1: start worker processes
2026/10/09 12:33:53 [notice] 1#1: start worker process 23
2026/10/09 12:33:53 [notice] 1#1: start worker process 24
192.168.168.10 - - [09/Oct/2026:12:35:40 +0000] "GET / HTTP/1.1" 200 16 "-" "curl/7.76.1" "-"
192.168.168.10 - - [09/Oct/2026:12:38:06 +0000] "GET / HTTP/1.1" 200 16 "-" "curl/7.76.1" "-"
```

### 3.3镜像优化

#### 3.3.1解决镜像安装源问题

```bash
#先下拉centos镜像
[root@docker ~]# docker pull centos:7
[root@docker ~]# docker images
                                                            i Info →   U  In Use
IMAGE            ID             DISK USAGE   CONTENT SIZE   EXTRA
busybox:latest   fd7dc98638c8       6.74MB         2.23MB
busybox:v1       9fee24b6b48f        237MB         63.9MB
centos:7         be65f488b776        299MB         76.1MB    U


#镜像存在问题
[root@docker dockerbuild]# docker run -it --rm centos:7 /bin/bash
[root@b5772cc4ce2a /]# yum install gcc -y
Loaded plugins: fastestmirror, ovl
Determining fastest mirrors
Could not retrieve mirrorlist http://mirrorlist.centos.org/?release=7&arch=x86_64&repo=os&infra=container error was
14: curl#6 - "Could not resolve host: mirrorlist.centos.org; Unknown error"
  
  
  
#解决镜像问题
[root@docker dockerbuild]# vim centos.repo
[centos]
name = centos7.9
baseurl = https://mirrors.aliyun.com/centos/7.9.2009/os/x86_64/
gpgcheck = 0
[root@docker dockerbuild]# vim Dockerfile
FROM   centos:7                          
RUN rm -rf  /etc/yum.repos.d/*
COPY   centos.repo  /etc/yum.repos.d/
[root@docker dockerbuild]# docker build -t centos-7.9:repo .
[root@docker dockerbuild]# docker run -it --rm centos-7.9:repo /bin/bash
[root@bb47ca1f7499 /]# yum install gcc -y
Complete!
```

#### 3.3.2利用现有惊现构建企业所需业务

```bash
[root@docker dockerbuild]# wget https://nginx.org/download/nginx-1.30.5.tar.gz
[root@docker dockerbuild]# vim Dockerfile
FROM   centos-7.9:repo                       #用上个实验构建出来的镜像
ADD    nginx-1.30.5.tar.gz  /mnt             
WORKDIR  /mnt/nginx-1.30.5
RUN    sed -i 's/CFLAGS="$CFLAGS -g"/#CFLAGS="$CFLAGS -g"/g' auto/cc/gcc
RUN    yum install gcc make pcre-devel openssl-devel -y
RUN    ./configure --with-http_ssl_module --with-http_stub_status_module
RUN    make && make install
EXPOSE 80
VOLUME ["/usr/local/nginx/html"]
CMD    ["/usr/local/nginx/sbin/nginx","-g","daemon off;"]
[root@docker dockerbuild]# docker build -t nginx-lee:v1 .
[root@docker dockerbuild]# docker images
                                                                    i Info →   U  In Use
IMAGE             ID             DISK USAGE   CONTENT SIZE   EXTRA
nginx-lee:v1      d95d7ce6a39f        476MB          123MB
[root@docker dockerbuild]# docker run -d --rm -p 80:80 nginx-lee:v1
17a9dc6545ac181a8461ebf24706ecc31645055326deb327ca1848321544cab2
[root@docker dockerbuild]# docker ps -a
CONTAINER ID   IMAGE          COMMAND                   CREATED          STATUS                  PORTS                                 NAMES
17a9dc6545ac   nginx-lee:v1   "/usr/local/nginx/sb…"   11 seconds ago   Up 11 seconds           0.0.0.0:80->80/tcp, [::]:80->80/tcp   interesting_lewin
[root@docker dockerbuild]# docker exec -it interesting_lewin /bin/bash
[root@17a9dc6545ac nginx-1.30.5]# /usr/local/nginx/sbin/nginx -v
nginx version: nginx/1.30.5                       #查看版本
```

### 3.5构建镜像优化方案

#### 3.5.1缩减镜像层

```bash
[root@docker dockerbuild]# vim Dockerfile            #减少RUN层，合并成一个
FROM   centos-7.9:repo         
ADD    nginx-1.30.5.tar.gz  /mnt
WORKDIR  /mnt/nginx-1.30.5
RUN    sed -i 's/CFLAGS="$CFLAGS -g"/#CFLAGS="$CFLAGS -g"/g' auto/cc/gcc &&  yum install gcc make pcre-devel openssl-devel -y && ./configure --with-http_ssl_module --with-http_stub_status_module && make && make install
EXPOSE 80
VOLUME ["/usr/local/nginx/html"]
CMD    ["/usr/local/nginx/sbin/nginx","-g","daemon off;"]
[root@docker dockerbuild]# docker build -t nginx-lee:v2 .
```

#### 3.5.2删除构建过程中的需要的临时文件

```bash
[root@docker dockerbuild]# vim Dockerfile
FROM   centos-7.9:repo            
ADD    nginx-1.30.5.tar.gz  /mnt
WORKDIR  /mnt/nginx-1.30.5
RUN    sed -i 's/CFLAGS="$CFLAGS -g"/#CFLAGS="$CFLAGS -g"/g' auto/cc/gcc &&  yum install gcc make pcre-devel openssl-devel -y && ./configure --with-http_ssl_module --with-http_stub_status_module && make && make install && rm -rf /mnt/* && yum clean all     #添加了清理命令
EXPOSE 80
VOLUME ["/usr/local/nginx/html"]
CMD    ["/usr/local/nginx/sbin/nginx","-g","daemon off;"]
[root@docker dockerbuild]# docker build -t  nginx-lee:v3 .
```

#### 3.5.3多阶构建

**在第一台主机编辑好以后复制到第二台主机，第一台主机不使用**

```bash
[root@docker dockerbuild]# vim Dockerfile
FROM   centos-7.9:repo   AS  build
ADD    nginx-1.30.5.tar.gz  /mnt
WORKDIR  /mnt/nginx-1.30.5
RUN    sed -i 's/CFLAGS="$CFLAGS -g"/#CFLAGS="$CFLAGS -g"/g' auto/cc/gcc &&  yum install gcc make pcre-devel openssl-devel -y && ./configure --with-http_ssl_module --with-http_stub_status_module && make && make install && rm -rf /mnt/* && yum  clean all
EXPOSE 80
VOLUME ["/usr/local/nginx/html"]
CMD    ["/usr/local/nginx/sbin/nginx","-g","daemon off;"]


FROM  centos:7
COPY --from=build /usr/local/nginx /usr/local/nginx
EXPOSE 80
VOLUME ["/usr/local/nginx/html"]
CMD  ["/usr/local/nginx/sbin/nginx","-g","daemon off;"]
[root@docker dockerbuild]# docker build -t nginx-lee:v4 .
[root@docker dockerbuild]# docker run -d --rm -p 80:80 nginx-lee:v4   #运行不了，80端口被占用
4b1be2a20a1499bb63709f5a723f6e9f97dd0e4e5ef919c610a7b29f00dcd7cf
docker: Error response from daemon: failed to set up container networking: driver failed programming external connectivity on endpoint stupefied_thompson (d52bdeae8cd91cc6dcc62a36d1a81dfc9673978f81fa932b4cc739bdae5616af): Bind for 0.0.0.0:80 failed: port is already allocated
[root@docker dockerbuild]# docker ps -a
CONTAINER ID   IMAGE          COMMAND                   CREATED             STATUS                  PORTS                                 NAMES
17a9dc6545ac   nginx-lee:v1   "/usr/local/nginx/sb…"   About an hour ago   Up About an hour        0.0.0.0:80->80/tcp, [::]:80->80/tcp   interesting_lewin
[root@docker dockerbuild]# docker rm -f 17a9dc6545ac    #删掉占用端口的镜像
17a9dc6545ac
[root@docker dockerbuild]# docker run -d --rm -p 80:80 nginx-lee:v4
86918aad87baa00fee37c8fa25c2b074e9f685f7feaef6168a8cc2f924665196  #可以正常运行
[root@docker ~]# docker exec heuristic_wright /usr/local/nginx/sbin/nginx -v
```

### 3.6利用debian 13构建nginx镜像

#### 1.导入debian 13到系统中

```bash
[root@docker ~]# docker load -i base-debian13.tar
Loaded image: gcr.io/distroless/base-debian13:latest
[root@docker ~]# docker search debian
[root@docker ~]# docker pull debian:latest         #需要使用科学上网软件
[root@docker ~]# docker images
                                                                    i Info →   U  In Use
IMAGE            ID             DISK USAGE   CONTENT SIZE   EXTRA
debian:latest   913f6706df59        184MB         52.6MB
```

#### 2.准备构建镜像时的素材

```bash
[root@docker ~]# mkdir build
[root@docker ~]# cd build/
[root@docker build]# wget https://nginx.org/download/nginx-1.30.5.tar.gz
```

#### 3.编写构建策略

```bash
[root@docker build]# vim Dockerfile
FROM debian:latest
ADD  nginx-1.30.5.tar.gz /mnt
WORKDIR /mnt/nginx-1.30.5
RUN  apt update && apt install -y ca-certificates && apt install -y build-essential libpc
re2-dev zlib1g zlib1g-dev libssl-dev libgd-dev libxml2-dev libxslt-dev && ./configure --w
ith-http_ssl_module --with-http_stub_status_module && make && make install
EXPOSE 80
VOLUME ["/usr/local/nginx/html"]
CMD    ["/usr/local/nginx/sbin/nginx","-g","daemon off;"]           
```

#### 4.构建镜像

```bash
[root@docker build]# docker build -t nginx-debian13:latest .
[+] Building 639.5s (9/9) FINISHED                                        docker:default
 => [internal] load build definition from Dockerfile                                0.0s
 => => transferring dockerfile: 468B                                                0.0s
 => [internal] load metadata for docker.io/library/debian:latest                    0.0s
 => [internal] load .dockerignore                                                   0.0s
 => => transferring context: 2B                                                     0.0s
 => [internal] load build context                                                   0.0s
 => => transferring context: 42B                                                    0.0s
 => [1/4] FROM docker.io/library/debian:latest@sha256:913f6706df59a68922d1dd08f78c  0.0s
 => => resolve docker.io/library/debian:latest@sha256:913f6706df59a68922d1dd08f78c  0.0s
 => CACHED [2/4] ADD  nginx-1.30.5.tar.gz /mnt                                      0.0s
 => CACHED [3/4] WORKDIR /mnt/nginx-1.30.5                                          0.0s
 => [4/4] RUN  apt update && apt install -y ca-certificates && apt install -y bu  614.3s
 => exporting to image                                                             25.0s
 => => exporting layers                                                            16.1s
 => => exporting manifest sha256:64578c029d48bffd0fa2ba1f8f6410e1dd4a2f0375fe37c17  0.3s
 => => exporting config sha256:8191ef59b6d432069f758b144eafff670c65e09089f6f0fc541  0.0s
 => => exporting attestation manifest sha256:5acf18507ba0c55b6fd784cad34c20bcbd3cd  0.0s
 => => exporting manifest list sha256:ec9f906fe70b468e89f3b8c07ceaabdc376b3d61275f  0.0s
 => => naming to docker.io/library/nginx-debian13:latest                            0.0s
 => => unpacking to docker.io/library/nginx-debian13:latest                         8.5s
[root@docker build]# docker images
                                                                    i Info →   U  In Use
IMAGE                     ID            DISK USAGE   CONTENT SIZE   EXTRA
nginx-debian13:latest   ec9f906fe70b        965MB          256MB
```

### 3.7构建最终镜像

#### 1.查看应用软件运行时所需要的库文件位置

```bash
[root@docker nginx]# docker run -d --name test -p 80:80 nginx-debian13:latest
cc3d32dc0f81e04b72a15b97b245aa279ccba32efe545340bfa032de20a1f0a8
[root@docker nginx]# docker exec -it test /bin/bash
root@cc3d32dc0f81:/mnt/nginx-1.30.5# ldd /usr/local/nginx/sbin/nginx
```

```bash
[root@docker ~]# mkdir build
[root@docker ~]# cd build/
[root@docker ~]# cd nginx/
[root@docker nginx]# vim Dockerfile
FROM nginx-debian13:latest AS build
RUN  mkdir -p /opt/var/cache/nginx && \          #先建一个空目录，留着给 nginx 放临时文件用
     cp -a --parents /etc/passwd /opt && \       #用户和用户组名册
     cp -a --parents /etc/group /opt  && \       #用户和用户组名册
     cp -a --parents /usr/local/nginx /opt && \    #nginx 本体
     cp -a --parents /usr/lib/x86_64-linux-gnu/libcrypt.so.1 /opt  && \ 
#密码加密相关
     cp -a --parents  /usr/lib/x86_64-linux-gnu/libpcre2-8.so.0 /opt && \ #正则表达式
     cp -a --parents  /usr/lib/x86_64-linux-gnu/libssl.so.3 /opt && \ #HTTPS 加密
     cp -a --parents  /usr/lib/x86_64-linux-gnu/libcrypto.so.3 /opt && \ #HTTPS 加密
     cp -a --parents  /usr/lib/x86_64-linux-gnu/libz.so.1 /opt && \      #压缩
     cp -a --parents  /usr/lib/x86_64-linux-gnu/libc.so.6 /opt && \      #最基础的 C 库，几乎所有程序都要
     cp -a --parents  /usr/lib/x86_64-linux-gnu/libzstd.so.1 /opt   

FROM gcr.io/distroless/base-debian13:latest
COPY --from=build /opt /
EXPOSE 80 443
VOLUME ["/usr/local/nginx/html"]
ENTRYPOINT ["/usr/local/nginx/sbin/nginx","-g","daemon off;"]
[root@docker nginx]# docker build -t nginx-lee:v1  . 
[root@docker nginx]# docker run -d --name webserver -p 80:80 nginx-lee:v1
0b04b1e979dcf8046c7732c2019abc3e9f7cc2fee7ed4ceec82e000650e1cb12
```

