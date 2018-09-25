## 发布系统搭建及问题解决

### 1 环境搭建

* nginx(./configure && make && make install)

由于nginx要使用正则表达式，因此，需要安装PCRE。

* php-fpm(PHP version:5.3.29)

```
./configure \
--enable-fpm \
--with-fpm-user=nginx \
--with-fpm-group=nginx \
--with-libxml-dir \
--with-gd \
--with-jpeg-dir \
--with-png-dir \
--with-zlib-dir \
--enable-gd-native-ttf \
--enable-gd-jis-conv \
--enable-mysqlnd \
--with-mysql=mysqlnd \
--with-mysqli=mysqlnd \
--with-pdo-mysql=mysqlnd \
--enable-sockets \
--enable-soap \
--with-curl \
--with-mcrypt \
--enable-mbstring
```

问题1：没有找到libiconv库

解决办法：下载libiconv并手动安装，为了避免对系统的库产生影响，这里在安装时指定安装目录。

./configure --prefix=/usr/local/libiconv && make && make install

同时，在编译php时，也需要指定iconv的安装路径：`--with-iconv=/usr/local/libiconv`

### 2 前端调试

配置好nginx和php-fpm，并启动后，就可以进行测试了，测试的第一步就是phpinfo()，这步可以正常打开。

然后直接打开我们自己的网站，发现无法打开，报404错误，看起来这里应该是nginx没有配置好。

问题2：nginx配置

此次关注的nginx配置中的两个地方：

* 使用location指令时，可以对URL进行正则匹配，需要注意这里的正则表达式
* 需要配置fastcgi_param

问题3：PHP时区的设定

环境搭建好后，用phpinfo()进行测试，提示有warning:

```
It is not safe to rely on the system's timezone settings. You are *required* to use the date...
```

从字面意思上的提示可以看出是时区设置的不对，然后系统会将时区设置为重庆。然后修改php.ini中的`date.timezone=Asia/Shanghai`，重启提示依旧，于是，不得已采用另一种方式：在index.php中添加`date_default_timezone_set("Asia/Shanghai")`或者`ini_set("date.timezone", "Asia/Shanghai")`。

问题4：鉴权

此时，打开网站时，发现有几个鉴权的失败，出现这种错误可能有两个原因：

1 无法解析目标机器

1 鉴权的目标机器不通

2 鉴权的模块有问题

发现可以ping通目标机器，但是访问时，是用域名访问的，而域名无法解析，修改/etc/hosts修复。

访问策略及其它：

* 1 各种DB的访问权限：自身的数据库、配置中心、密码库
* 2 rsync：需要将文件传送到该机器，需要开通rsync策略。

问题5：rsync认证失败

如果采用模块的方式，则需要将密码文件的权限设置为600。另外，使用模块的方式既可以使用系统已经有的用户，也可以使用自己定义的用户，而采用非模块的方式，就只能使用系统已有的用户。

问题6：source的问题

source的功能：在当前shell环境下执行脚本，与`.`相同。而该命令在查找规则是会在PATH和当前路径中查找中进行查找，因此，在bash中，如果是以绝对路径调用一个脚本，而这个脚本会调用它所在目录中的文件，但是当前又不在这个目录中，就会报`No such file or directory`，因为source只会在PATH和当前路径中查找。

而且，shell不同，查找规则也可能不同，我所遇到的就是，机器上的shell默认不是bash，source在查找时只会查找PATH路径，而不会查找当前路径。因此，在写shell头部时，应该使用`#!/usr/bin/env bash`。

问题7：文件上传的限制

nginx的限制：

```
client_max_body_size 120m;
```

php.ini的限制：

```
post_max_size = 125M
upload_max_filesize = 120M
max_execution_time=90
```

问题8：php缺少扩展(例如mbstring)

方法一：

如果在编译php之前就知道用哪些扩展，在编译时，就可以加上这些扩展选项。例如，如果确认使用nginx作为服务器，那么就需要带上--enable-fpm。因此，如果已经确认要使用mbstring，那么可以在编译php时，就添加–-enable-mbstring。

方法二：

首先编译扩展：

```
cd /usr/src/php-5.3.29/ext/mbstring
/usr/local/php/bin/phpize
./configure --with-php-config=/usr/local/php/bin/php-config
make && make install
```

然后重启web服务。

方法三：

安装php-mbstring：`yum install php-mbstring`；然后修改php.ini，添加：`extension=mbstring.so`；再重启web服务即可。

* python mysql、DBUtils

问题9：升级python并安装MySQLdb

由于机器上的python版本是2.6的，而服务器常用的版本是2.7，而且由于用到了2.7版本的OrderedDict，于是，需要将python升级到2.7。
下载python2.7的包，然后./configure && make && make install，由于有多个版本共存，此时，需要将系统的python链接指向python2.7：

```
mv /usr/bin/python /usr/bin/python2.6.6
ln -s /usr/local/bin/python2.7 /usr/bin/python
```

然后安装setuptools。

下载MySQLdb的包并安装，安装该包时，可能会找不到mysql.h，此时需要安装mysql的开发库(mysql-devel.x86_64)，可能会找不到mysql_config，需要将mysql_config所在的目录添加到PATH路径中，另外还要安装python的开发包：python-devel.x86_64，否则会出现下列错误：

```
pymemcompat.h:10:20: error: Python.h No such file or directory
```

其它：db的搭建、配置以及导入

将前端网站、后台SVR搭建完成后，下一步就是DB的搭建和配置，最后就是很熟悉的一个操作：数据库的导出和导入：

1 数据库的导出：

```
mysqldump -uuser_name -p database_name [table_name] > database_name.sql
```

2 数据库的导入：

可以在mysql中执行`source database_name.sql`，也可以在命令行中执行`mysql -uuser_name -p database_name < database_name.sql`。