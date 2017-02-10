## 系统迁移搭建过程中遇到的问题

* PHP时区的设定

环境搭建好后，用phpinfo()进行测试，提示有warning:

```
It is not safe to rely on the system's timezone settings. You are *required* to use the date...
```

从字面意思上的提示可以看出是时区设置的不对，然后系统会将时区设置为重庆。然后修改php.ini中的`date.timezone=Asia/Shanghai`，重启提示依旧，于是，不得已采用另一种方式：在index.php中添加`ini_set("date.timezone", "Asia/Shanghai")`。

* 鉴权

修改完时区后，在浏览器中访问，打不开，然后查看PHP的日志，发现日志中有很多鉴权的错误，出现这种错误可能有两个原因：

1 鉴权的目标机器不通

2 鉴权的模块有问题

通过与同事沟通，然后ping鉴权的目标机器，发现是目标机器不通，申请访问策略后就OK了。

* db的搭建、配置以及导入

将前端网站、后台SVR搭建完成后，下一步就是DB的搭建和配置，最后就是很熟悉的一个操作：数据库的导出和导入：

1 数据库的导出：

```
mysqldump -uuser_name -p database_name [table_name] > database_name.sql
```

2 数据库的导入：

可以在mysql中执行`source database_name.sql`，也可以在命令行中执行`mysql -uuser_name -p database_name < database_name.sql`。

* source的问题

source的功能：在当前shell环境下执行脚本，与`.`相同。而该命令在查找规则是会在PATH和当前路径中查找中进行查找，因此，在bash中，如果是以绝对路径调用一个脚本，而这个脚本会调用它所在目录中的文件，但是当前又不在这个目录中，就会报`No such file or directory`，因为source只会在PATH和当前路径中查找。

而且，shell不同，查找规则也可能不同，我所遇到的就是，机器上的shell默认不是bash，source在查找时只会查找PATH路径，而不会查找当前路径。因此，在写shell头部时，应该使用`#!/usr/bin/env bash`。

* 文件上传的限制

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

* php缺少扩展(例如mbstring)

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