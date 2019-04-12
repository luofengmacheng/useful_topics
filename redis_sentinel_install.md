## redis sentinel高可用部署及用python访问

### 1 redis sentinel高可用部署

下载好对应版本的redis压缩包后，直接make就可以编译成可执行程序。

这里部署的高可用采用一主两备的方式，并且用sentinel自动切换master。

首先安装redis。

修改redis.conf配置文件：

```
bind $local_ip

daemonize yes

# 如果是备需要配置slaveof
slaveof $master_host
```

然后可以启动redis集群：

```
./redis-server ../redis.conf
```

开始安装sentinel。

修改sentinel.conf配置文件(三台机器用一样的配置)：

```
bind $local_ip

daemonize yes

sentinel monitor mymaster $master_host 6379 2
```

然后启动sentinel集群：

```
./redis-sentinel ../sentinel.conf
```

然后就可以通过客户端链接sentinel查看状态。

### 2 使用python访问redis sentinel

``` python
#!/usr/bin/env python

import redis
from redis.sentinel import Sentinel

sentinel = Sentinel([
    ('100.117.98.3', 26379),
    ('100.117.98.5', 26379),
    ('100.117.98.6', 26379),
], socket_timeout=0.5)

master = sentinel.discover_master('mymaster')
print master

slaves = sentinel.discover_slaves('mymaster')
print slaves

master = sentinel.master_for('mymaster', socket_timeout=0.5)
w_ret = master.set('luo', 'feng')

slave = sentinel.slave_for('mymaster', socket_timeout=0.5)
r_ret = slave.get('luo')
```