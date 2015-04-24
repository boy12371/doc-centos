# Install mysql2redis on CentOS6

使用mysql2redis可以非常便捷的将mysql中的数据导出到redis中去, 通常是需要一个select语句即可实现。作者号称：In my poor labtop, 10,000 row data were synced into redis in 0.4 second。在centos6安装mysql2redis，先安装mysql5.6，[免费下载](http://download.softagency.net/MySQL/Downloads/)

* 1、安装jemalloc(内存管理)
```shell
$ yum -y install epel-release
$ yum -y install jemalloc jemalloc-devel
```

* 2、安装apr(Apache Portable Runtime libraries)
```shell
$ wget http://mirror.bit.edu.cn/apache//apr/apr-1.5.1.tar.gz
$ tar -zxvf apr-1.5.1.tar.gz
$ cd apr-1.5.1
$ ./configure
$ make && make install
```

* 3、安装apr-util(同上)
```shell
$ wget http://mirror.bit.edu.cn/apache//apr/apr-util-1.5.4.tar.gz
$ tar -zxvf apr-util-1.5.4.tar.gz
$ cd apr-util-1.5.4
$ ./configure --with-apr=/usr/local/apr
$ make && make install
```

* 4、安装lib_mysqludf_json(mysql的json插件)，编译时必须依赖mysql-devel
```shell
$ wget https://github.com/mysqludf/lib_mysqludf_json/archive/master.zip
$ unzip master.zip
$ cd lib_mysqludf_json-master
// 其中的lib_mysqludf_json.so是32位的，64位下必须重新编译
$ rm -rf lib_mysqludf_json.so
$ gcc $(mysql_config --cflags) -shared -fPIC -o lib_mysqludf_json.so lib_mysqludf_json.c
$ cp lib_mysqludf_json.so /usr/lib64/mysql/plugin/
```

* 5、安装hiredis(redis的C语言client)
```shell
$ wget https://github.com/redis/hiredis/archive/v0.12.1.zip
$ unzip v0.12.1.zip
$ cd hiredis-0.12.1/
$ make && make install
```

* 6、安装mysql2redis(mysql的导出数据到redis插件)，目前编译时只支持hiredisv0.12.1
```shell
$ wget https://github.com/dawnbreaks/mysql2redis/archive/master.zip
$ unzip master.zip
$ cd mysql2redis-master/
$ make
$ cp lib_mysqludf_redis_v2.so /usr/lib64/mysql/plugin/
$ cat >> /etc/profile << EOF
$ LD_LIBRARY_PATH=/usr/local/apr/lib/:/usr/local/lib/:$LD_LIBRARY_PATH
$ export LD_LIBRARY_PATH
$ EOF
$ source /etc/profile
$ ldconfig -n $LD_LIBRARY_PATH
```

* 7、注册lib_mysqludf_json UDF
```shell
$ mysql -uroot -p$MYSQL_PASSWORD --execute="drop function lib_mysqludf_json_info;drop function json_array;drop function json_members;drop function json_object;drop function json_values;"
$ mysql -uroot -p$MYSQL_PASSWORD --execute="create function lib_mysqludf_json_info returns string soname 'lib_mysqludf_json.so';create function json_array returns string soname 'lib_mysqludf_json.so';create function json_members returns string soname 'lib_mysqludf_json.so';create function json_object returns string soname 'lib_mysqludf_json.so';create function json_values returns string soname 'lib_mysqludf_json.so';"
```

* 8、注册mysql2redis UDF
```shell
$ mysql -uroot -p$MYSQL_PASSWORD --execute="DROP FUNCTION IF EXISTS redis_servers_set_v2;DROP FUNCTION IF EXISTS redis_command_v2;DROP FUNCTION IF EXISTS free_resources;"
$ mysql -uroot -p$MYSQL_PASSWORD --execute="CREATE FUNCTION redis_servers_set_v2 RETURNS int SONAME 'lib_mysqludf_redis_v2.so';CREATE FUNCTION redis_command_v2 RETURNS int SONAME 'lib_mysqludf_redis_v2.so';CREATE FUNCTION free_resources RETURNS int SONAME 'lib_mysqludf_redis_v2.so';"
$ mysql -uroot -p$MYSQL_PASSWORD --execute="select * from mysql.func;"
$ mysql -uroot -p$MYSQL_PASSWORD --execute="show variables like '%plugin%';"
```
---
    author: 348350530(AT)qq.com