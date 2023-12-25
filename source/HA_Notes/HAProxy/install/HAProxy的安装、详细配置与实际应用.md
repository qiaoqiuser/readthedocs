# HAProxy的安装、详细配置与实际应用（MyCAT、RabbitMQ示例）             

 原文：https://juejin.cn/post/7150911359272615944?searchId=20231225140846B1C19B8DBF4EA19B6CE3           

## HAProxy概述

> HAProxy（High Availability Proxy）是一款自由、快速、可靠的TCP/HTTP负载均衡软件，其最常见的用途是将客户端请求分发到多个服务器上，从而实现高并发和高可用性。

> HAProxy实现一种事件驱动, 单一进程模型，此模型支持非常大的并发连接数。因此，适用于负载特大的web站点，运行在硬件上，完全可以支持数以万计的并发连接。

HAProxy官网：`http://www.haproxy.org/`

HAProxy文档：`http://docs.haproxy.org/`

## 下载

下载地址：`https://src.fedoraproject.org/repo/pkgs/haproxy/`

```

wget https://src.fedoraproject.org/repo/pkgs/haproxy/haproxy-2.6.0.tar.gz/sha512/7bb70bfb5606bbdac61d712bc510c5e8d5a5126ed8827d699b14a2f4562b3bd57f8f21344d955041cee0812c661350cca8082078afe2f277ff1399e461ddb7bb/haproxy-2.6.0.tar.gz
```

解压

```

tar -zxvf haproxy-2.6.0.tar.gz
```

## 编译

查看内核版本

```

[root@administrator haproxy-2.6.0]# uname -r
3.10.0-1160.62.1.el7.x86_64
```

开始编译

```

make TARGET=linux310 PREFIX=/usr/local/program/haproxy ARCH=x86_64
```

`ARGET=linux310`：内核版本

`ARCH=x86_64`：系统位数

`PREFIX=/usr/local/program/haproxy`：haprpxy安装路径

## 安装

```

make install PREFIX=/usr/local/program/haproxy
```

## 配置

建一个名为haproxy的用户组

```

# -r 将该用户组设置为系统级别的组
groupadd -r haproxy
```

创建一个名为haproxy的用户

```

# -g 将该用户添加到上面创建的"haproxy"用户组中
# -r 将该用户设置为系统级别用户
useradd -r -g haproxy haproxy
```

注意: `让haproxy使用haproxy用户和组运行，提高安全性`

创建HAProxy配置文件，具体配置可参考`/haproxy-2.6.0/examples/`目录下的配置文件，如：`quick-test.cfg`

```

vim /usr/local/program/haproxy/haproxy.conf
```

**这里配置HAProxy实现MyCat多节点的集群高可用和负载均衡为例**

> 若要HAProxy自身高可用则可以通Keepalived来实现

```

# 全局配置参数，属于进程级的配置
global
    # 日志配置 local0:日志设备 info:日志记录级别
    log 127.0.0.1 local0 info
    # haproxy工作目录
    chroot /usr/local/program/haproxy
    # haproxy启动后进程的pid文件路径
    pidfile /usr/local/program/data/haproxy.pid
    # 每个haproxy进程可接受的最大并发连接数
    maxconn 4000
    # 使用haproxy用户和haproxy组运行
    user    haproxy        
    group   haproxy  
    # haproxy启动时可创建的进程数,默认1个，值应小于服务器的CPU核数
    # 该参数在高版本种已弃用
    nbproc  1
    # 以后台守护进程方式启动haproxy
    daemon
    
# 定义默认配置选项   
defaults
	# 使用http代理模式
    mode tcp
    # 开启全局日志记录
    log global
    option abortonclose
    option redispatch
    # 配置连接后端服务器失败重试次数，超过3次后会将失败的后端服务器标记为不可用
    retries 3
    # 连接超时时间 配置成功连接到一台服务器的最长等待时间，默认单位是毫秒，也可自己指定单位
    timeout connect 10000
    # 客户端连接超时时间 配置连接客户端发送数据时的最长等待时间，默认单位是毫秒，也可自己指定单位
    timeout client 1m
    # 后端服务器响应超时时间 配置服务器端回应客户端数据发送时最长等待时间，默认单位是毫秒，也可自己指定单位
    timeout server 1m
    # 配置对后端服务器的检测超时时间，默认单位是毫秒，也可自己指定单位
    timeout check  10s
    # 最大连接数
    maxconn 3000
    
# 定义服务叫"proxy_status "名字的虚拟节点
# haproxy代理的两个mycat
listen proxy_status 
	# 配置监听8086端口
    bind  0.0.0.0:8086
    	  # tcp模式  
          mode tcp
          # 负载均衡算法，使用轮询方式分发请求, 轮询访问mycat_1与mycat_2  
          # 轮询算法：roundrobin 权重算法：static-rr 最少连接算法：leastconn 请求源IP算法：source   
          balance roundrobin
          # 后端服务器列表, mycat真实IP:端口
          server mycat_1 IP:8066 check inter 10s
          server mycat_2 IP:8066 check inter 10s


# 定义服务叫"admin_stats"名字的虚拟节点     
# haproxy管理页面
frontend admin_stats 
    # 监听地址和端口 绑定所有可用的IP地址和端口8085
    bind  *:8085
          # http模式  
          mode http
          # 配置在客户端和服务器完成一次连接请求后，haproxy主动关闭此TCP连接
          option httpclose
          # 配置后端服务器需要获得客户端的真实IP，通过增加"X-Forwarded-For"来记录客户端IP
          option forwardfor
          # 启用日志来记录http请求，默认只对tcp日志进行日志记录
          option httplog
          maxconn 10
          stats enable
          stats refresh 30s
          # 统计页面路径
          stats uri /admin
          # 设置统计页面认证的用户和密码
          stats auth admin:123123
          stats hide-version
          stats admin if TRUE
```

## 启动

**1.启动HAProxy**

```

 /usr/local/program/haproxy/sbin/haproxy -f /usr/local/program/haproxy/haproxy.conf
```

启动HAProxy出现异常：

```

[root@administrator haproxy]# /usr/local/program/haproxy/sbin/haproxy -f /usr/local/program/haproxy/haproxy.conf
[NOTICE]   (2371) : haproxy version is 2.6.0-a1efc04
[NOTICE]   (2371) : path to executable is /usr/local/program/haproxy/sbin/haproxy
[ALERT]    (2371) : config : parsing [/usr/local/program/haproxy/haproxy.conf:39]: Missing LF on last line, file might have been truncated at position 32.
[ALERT]    (2371) : config : Error(s) found in configuration file : /usr/local/program/haproxy/haproxy.conf
[ALERT]    (2371) : config : Fatal errors found in configuration.
```

原因：

```

1.换行符导致，删掉换行符

2.使用文本编辑器修改该文件导致，使用`vim haproxy.conf`方式编辑保存
```

**2.查看HAProxy进程**

```

ps -ef|grep haproxy

[root@administrator haproxy]# ps -ef|grep haproxy
root      5186     1  0 14:06 ?        00:00:00 /usr/local/program/haproxy/sbin/haproxy -f /usr/local/program/haproxy/haproxy.conf
root      9133  5733  0 14:08 pts/0    00:00:00 grep --color=auto haproxy
```

停止haproxy

```

[root@administrator haproxy]# kill -9 5186     
```

## 验证

**1.打开浏览器访问**

> 浏览器访问：`http://IP:8085/admin`，输入配置的用户名与密码

![在这里插入图片描述](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/603bb1ce0ce24b11a930fa3de201ea27~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

**2.验证**

> 通过HAProxy访问Mycat，只需要访问配置HAProxy的`8086`端口即可，HAProxy将自动连接Mycat。

执行命令：`mysql -umycat -p123456 -h IP -P 8086`

```

[root@administrator ~]# mysql -umycat -p123456 -h IP -P 8086
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.6.29-mycat-1.6.7.6-release-20220524173810 MyCat Server (OpenCloudDB)

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```

## RabbitMQ的HAProxy配置示例

> 额外配置HAProxy实现RabbitMQ多节点的集群高可用和负载均衡示例

```

# 全局配置参数，属于进程级的配置
global
	
    
# 默认参数配置    
defaults

    
# 定义服务叫"proxy_status "名字的虚拟节点
# haproxy代理的两个mq
listen proxy_status 
	# 配置监听5672端口
    bind  0.0.0.0:5672
    	  # tcp模式  
          mode tcp
          # 轮询访问mq1与mq2
          balance roundrobin
          # mq真实IP:端口
          server node01 IP:5673 check inter 10s
          server node02 IP:5674 check inter 10s

  
# haproxy管理页面
frontend admin_stats 
    # 监听地址和端口
    bind  *:8888
          # http模式  
          mode http
```

访问`http://192.168.10.13:8888/admin`查看HAProxy监控 ![在这里插入图片描述](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bc6b2d7d573f4c1a8da565b9ebb51a7f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)