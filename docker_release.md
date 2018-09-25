## Docker构建php、nginx、python环境

### 1 运行镜像并进入容器

```
docker run –d --net host docker.oa.com:8080/gaia/tlinux2-64bit
docker exec -it CONTAINER_ID /bin/bash
```

### 2 部署nginx、php和python

进入容器后，安装以下这些软件，由于使用了主机的网络，因此，可以采用yum安装软件需要的组件。

* nginx
* php & phpredis
* python & setuptools & MySQL-python & DBUtils

```
docker commit -m "XXX" CONTAINER_ID docker.oa.com:8080/g_lovellluo/pkgrelease:v1.0
```

### 3 构建镜像

* build时报`invalid id:`，建议尝试使用不同版本的docker

编写好Dockerfile后，就可以进行构建了：

```
docker build -t docker.oa.com:8080/g_lovellluo/pkgrelease:v2.0 .
```

### 4 保存及导入镜像

保存镜像

```
docker save -o pkgrelease.tar docker.oa.com:8080/g_lovellluo/pkgrelease:v2.0
```

载入镜像

```
docker load -i pkgrelease.tar
```