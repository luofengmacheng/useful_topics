## 安装docker

### 1 系统要求

安装docker一般需要操作系统满足以下的条件：
* 操作系统版本较新
* 64位

### 2 进行安装

yum进行安装：

```
yum install docker
```

安装后就可以使用`docker version`查看版本号。

启动docker服务：

```
systemctl start docker
```

### 3 配置docker

不同的linux发行版使用的配置文件不太一样，例如：

ubuntu 14.04使用/etc/init/docker.conf，ubuntu 15.04使用/etc/default/docker，
而centos 6.x使用/etc/sysconfig/docker，docker 1.11后则默认没有创建配置文件，需要手工创建。

使用下面的命令可以查看docker daemon使用了哪个配置文件，然后去修改配置文件。

```
systemctl show docker | grep Environment
```

如果没有使用任何配置文件，可以自行创建：

```
touch /etc/systemd/system/docker.service.d/docker.conf
```

并写入以下内容，并可以添加启动参数，例如无TLS认证的仓库地址：

```
[Service]  
ExecStart=  
ExecStart=/usr/bin/docker daemon --insecure-registry=XXXX
```

### 4 使用systemctl控制服务

* systemctl restart docker 重启docker服务
* systemctl enable docker 开机自启动docker服务
* systemctl show docker 查看docker服务启动的参数配置
