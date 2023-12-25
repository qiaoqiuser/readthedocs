#             Haproxy搭建web群集              

原文：https://juejin.cn/post/7169552758167568392?searchId=20231225140846B1C19B8DBF4EA19B6CE3            

# 一、常见的Web集群调度器

**目前常见的Web集群调度器分为软件和硬件：**

- 软件通常使用开源的LVS、Haproxy、 Nginx
- LVS性能最好，但是搭建相对复杂；Nginx 的upstream模块支持群集功能，但是对群集节点健康检查功能不强，高并发性能没有Haproxy好。
- 硬件一般使用比较多的是F5、Array，也有很多人使用国内的一些产品，如梭子鱼、绿盟等
- 硬件的效果比软件好，更加稳定，但管理成本高。

# 二、Haproxy 应用分析

**LVS在企业应用中抗负载能力很强，但存在不足**

- LVS不支持正则处理，不能实现动静分离
- 对于大型网站，LVS的实施配置复杂，维护成本相对较高

**Haproxy是一款可提供高可用性、负载均衡、及基于TCP和HTTP应用的代理的软件**

- 适用于负载大的Web站点
- 运行在硬件上可支持数以万计的并发连接的连接请求

# 三、HAProxy的特性

HAProxy是可提供高可用性、负载均衡以及基于TCP和HTTP应用的代理，是免费、快速并且可靠的一种解决方案。HAProxy非常适用于并发天（并发达1w以上）web站点，这些站点通常又需要会话保持或七层处理。HAProxy的运行模式使得它可以很简单安全的整合至当前的架构中，同时可以保护web服务器不被暴露到网络上。

### HAProxy的主要特性有：

- 可靠性和稳定性非常好，可以与硬件级的F5负载均衡设备相媲美；
- 最高可以同时维护40000-50000个并发连接，单位时间内处理的最大请求数为20000个，最大处理能力可达10Git/s;
- 支持多达8种负载均衡算法；
- 支持Session会话保持，Cookie的引导；
- 支持通过获取指定的url来检测后端服务器的状态；
- 支持虚机主机功能，从而实现web负载均衡更加灵活；
- 支持连接拒绝、全透明代理等独特的功能；
- 拥有强大的ACL支持，用于访问控制；
- 支持TCP协议的负载均衡转发；（支持7层代理、实现动静分离）
- 支持客户端的keepalive功能，减少客户端与haproxy的多次三次握手导致资源浪费，让多个请求在一个tcp连接中完成

# 四、Haproxy调度算法

## 1、RR（Round Robin）

RR算法是最简单最常用的一种算法，即轮询调度。

**理解举例：**

- 有三个节点A、B、C
- 第一个用户访问会被指派到节点A
- 第二个用户访问会被指派到节点B
- 第三个用户访问会被指派到节点C
- 第四个用户访问继续指派到节点A，轮询分配访问请求实现负载均衡 效果

## 2、static-rr

加权轮询，表示根据web服务器的权重来分配前端请求。

为web服务器设置不同权重，哪台服务器配置好，权重就多一点。

类似于Nginx负载局哼算法的ip_hash。

## 3、LC（Least Connections)

最小连接数算法，根据后端的节点连接数大小动态分配前端请求。

**理解举例：**

- 有三个节点A、B、C,各节点的连接数分别为A:4、B:5、C:6
- 第一个用户连接请求，会被指派到A上，连接数变为A:5、B:5、 C:6
- 第二个用户请求会继续分配到A上，连接数变为A:6、B:5、 C:6; 再有新的请求会分配给B，每次将新的请求指派给连接数最小的客户端
- 由于实际情况下A、B、C的连接数会动态释放，很难会出现-样连接数的情况
- 此算法相比较rr算法有很大改进，是目前用到比较多的一-种算法

## 4、SH (Source Hashing)

基于来源访问调度算法，用于一些有Session会话记录在服务器端的场景，可以基于来源的IP、Cookie等做集群调度。

**理解举例：**

- 有三个节点A、B、C,第一个用户第一次访问被指派到了A,第二个用户第一次访问被指派到了B。
- 当第一个用户第二次访问时会被继续指派到A,第二个用户第= _次访问时依旧会被指派到B，只要负载均衡调度器不重启，第-个用户访问都会被指派到A,第二个用户访问都会被指派到B，实现集群的调度。
- 此调度算法好处是实现会话保持，但某些IP访问量非常大时会引|起负载不均衡，部分节点访问量超大，影响业务使用。

## 5、uri

目的地址哈希，表示根据用户请求的URI做hash，做cdn时需要使用。（cdn：内容分发网络 ，边缘网络缓存加速）

url_hash 就是根据虚拟主机名称后面的目录。

## 6、url_ param

表示根据请求的URl参数'balance url_ param' requires an URL parameter name

## 7、hdr (name)

表示根据HTTP请求头来锁定每一次HTTP请求。

## 8、rdp-cookie (name)

表示根据cookie (name)来锁定并哈希每一次TCP请求。

# 五、haproxy提供了3种实现会话保持的方式

（1）源地址hash

（2）设置cookie

（3）会话粘性表stick-table

# 六、LVS、Nginx、HAproxy的区别

- LVS基于Linux操作系统内核实现软负载均衡，而HAProxy和Nginx是基于第三方应用实现的软负载均衡;
- LVS是可实现4层的IP负载均衡技术，无法实现基于目录、URL的转发。而HAProxy 和Nginx都可以实现4层和7层技术，HAProxy可提供TCP和HTTP应用的负载均衡综合解决方案；
- LVS因为工作在ISO模型的第四层，其状态监测功能单一，而HAProxy在状态监测方面功能更丰富、强大，可支持端口、URL、脚本等多种状态检测方。
- HAProxy功能强大，单纯从效率上来讲HAProxy会比Nginx有更出色的负载均衡速度，在并发处理上也是优于Nginx的。但整体性能低于4层模式的LVS负载均衡；
- Nginx主要用于web服务器或缓存服务器。Nginx的upstream模块虽然也支持群集功能，但是对群集节点健康检查功能不强，性能没有Haproxy好。

# 七、使用Haproxy搭建web群集

**实验环境：**

Haproxy负载均衡器：192.168.10.11/24

Nginx1：192.168.10.20/24

Nginx2：192.168.10.30/24

客户端：宿主机

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/26670ff349c04b06aadf79078e40c143~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

## 1.关闭防火墙，将安装Haproxy所需软件包传到/opt目录下

```

systemctl stop firewalld
setenforce 0
 
haproxy-1.5.19.tar.gz
```

## 2、编译安装 Haproxy

```

yum install -y pcre-devel bzip2-devel gcc gcc-c++ make
 
tar zxvf haproxy-1.5.19.tar.gz
cd haproxy-1.5.19/
make TARGET=linux2628 ARCH=x86_64
make install
```

---------------------参数说明---------------------------------------------------------------------------
 TARGET=linux26 #内核版本，
 \#使用uname -r查看内核，如：2.6.18-371.el5，此时该参数用TARGET=linux26；kernel大于2.6.28的用TARGET=linux2628

ARCH=x86_64 #系统位数，64位系统

## 3、Haproxy服务器配置

```

mkdir /etc/haproxy
cp examples/haproxy.cfg /etc/haproxy/
 
cd /etc/haproxy/
vim haproxy.cfg
global
--4~5行--修改，配置日志记录，local0为日志设备，默认存放到系统日志
        log /dev/log   local0 info     
        log /dev/log   local0 notice
        #log loghost    local0 info
        maxconn 4096                    #最大连接数，需考虑ulimit -n限制
--8行--注释，chroot运行路径，为该服务自设置的根目录，一般需将此行注释掉
        #chroot /usr/share/haproxy
        uid 99                          #用户UID
        gid 99                          #用户GID
        daemon                          #守护进程模式
 
defaults       
        log     global                  #定义日志为global配置中的日志定义
        mode    http                    #模式为http
        option  httplog                 #采用http日志格式记录日志
        option  dontlognull             #不记录健康检查日志信息
        retries 3                       #检查节点服务器失败次数，连续达到三次失败，则认为节点不可用
        redispatch                      #当服务器负载很高时，自动结束当前队列处理比较久的连接
        maxconn 2000                   #最大连接数，“defaults”中的值不能超过“global”段中的定义
        #contimeout 5000               #设置连接超时时间，默认单位是毫秒
        #clitimeout 50000               #设置客户端超时时间，默认单位是毫秒
        #srvtimeout 50000              #设置服务器超时时间，默认单位是毫秒
        timeout http-request 10s       #默认http请求超时时间
        timeout queue 1m               #默认队列超时时间
        timeout connect 10s            #默认连接超时时间，新版本中替代contimeout，该参数向后兼容
        timeout client 1m             #默认客户端超时时间，新版本中替代clitimeout，该参数向后兼容
        timeout server 1m             #默认服务器超时时间，新版本中替代srvtimeout，该参数向后兼容
        timeout http-keep-alive 10s   #默认持久连接超时时间
        timeout check 10s             #设置心跳检查超时时间
 
--删除下面所有listen项--，添加
listen  webcluster 0.0.0.0:80           #定义一个名为webcluster的应用
        option httpchk GET /test.html   #检查服务器的test.html文件
        balance roundrobin              #负载均衡调度算法使用轮询算法roundrobin
        server inst1 192.168.10.20:80 check inter 2000 fall 3      #定义在线节点
        server inst2 192.168.10.30:80 check inter 2000 fall 3
        
---------------------参数说明---------------------------------------------------------------------------
balance roundrobin      #负载均衡调度算法
#轮询算法：roundrobin；最小连接数算法：leastconn；来源访问调度算法：source，类似于nginx的ip_hash
 
check inter 2000        #表示haproxy服务器和节点之间的一个心跳频率
fall 3                  #表示连续三次检测不到心跳频率则认为该节点失效
若节点配置后带有“backup”表示该节点只是个备份节点，只有主节点失效该节点才会上。不携带“backup”，表示为主节点，和其它主节点共同提供服务。
```

## 4、添加haproxy 系统服务

```

cp /opt/haproxy-1.5.19/examples/haproxy.init /etc/init.d/haproxy
chmod +x haproxy
chkconfig --add /etc/init.d/haproxy
 
ln -s /usr/local/sbin/haproxy /usr/sbin/haproxy
service haproxy start   或   /etc/init.d/haproxy start
```

## 5、节点服务器部署

```

systemctl stop firewalld
setenforce 0
 
yum install -y pcre-devel zlib-devel gcc gcc-c++ make
 
useradd -M -s /sbin/nologin nginx
 
cd /opt
tar zxvf nginx-1.12.2.tar.gz -C /opt/
 
cd nginx-1.12.2/
./configure --prefix=/usr/local/nginx --user=nginx --group=nginx && make && make install
 
make && make install
 
--192.168.10.20---
echo "this is zj web" > /usr/share/nginx/html/index.html
 
--192.168.10.30---
echo "this is zhou web" > /usr/share/nginx/html/index.html
 
ln -s /usr/share/nginx/sbin/nginx /usr/local/sbin/
 
nginx      #启动nginx 服
 
或者用yum安装
cat > /etc/yum.repos.d/nginx.repo << 'EOF'
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/7/$basearch/
gpgcheck=0
EOF
 
yum install nginx -y
```

## 6、测试 Web群集

在客户端使用浏览器打开 [http://192.168.10.11/index.html](https://link.juejin.cn?target=http%3A%2F%2F192.168.10.11%2Findex.html) ，不断刷新浏览器测试负载均衡效果

# 六、实例操作：Haproxy集群的构建

## 1、关闭防火墙，将安装Haproxy所需软件包传到/opt目录下

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d310a273ebaa4110a2cb0a234d708036~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

## 2、编译安装 Haproxy

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/87425a0ff2074bc49748bc702f59bc73~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/339eb72b68e446eda0e43c7cdbdbcc8e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb2600dee22e44f3bc738e4677d29448~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/44d29ae656cf43cbb71cd01f64fda7de~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/072ca8baa8fe4cd5b66bb94bd7b982e0~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

## 3、Haproxy服务器配置

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4aa48516e6de4637bf80daf19859fe90~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/09c7daad669a4ab092d61d69cf1b6ff3~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

## 4、添加haproxy 系统服务

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0c1780d6f4a348908604c4185d06d981~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9b6c8ab3b4314cff9da2bc91c01da38d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

## 5、节点服务器部署

Nginx 服务器1：192.168.10.20（编译安装）

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a036b36ca805419ba2b5d64ef4ff1570~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cef69856ab304efdbb731282c26918c4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d538c0a2377c4f7ca416e1b5bb0eb31e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e5668cca2e7d4156b8789a04e4a45a50~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/87a221fd5f0b431480c8c14406718727~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) Nginx 服务器2：192.168.10.30（yum安装）

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1715a5078127434c9e58fcd0c0e10c6a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1c10fb4b89034fe1bce17824fc16a3d2~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fa49d57b208b402482b4192d9fab0b6e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e6c878cfea8340b8a5947346eb36dcb3~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

## 6、测试 Web群集

在客户端使用浏览器打开[http://192.168.10.11/index.html，不断刷新浏览器测试负载均衡效果](https://link.juejin.cn?target=http%3A%2F%2F192.168.10.11%2Findex.html%EF%BC%8C%E4%B8%8D%E6%96%AD%E5%88%B7%E6%96%B0%E6%B5%8F%E8%A7%88%E5%99%A8%E6%B5%8B%E8%AF%95%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1%E6%95%88%E6%9E%9C)

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/27cc2e9028f9431d9c9072e697b7c760~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/338181d75a2346c1a23cabfb583b71b5~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

# 七、Haproxy集群的日志重新定义的操作步骤

默认haproxy的日志是输出到系统的syslog中，查看起来不是非常方便，为了更好的管理haproxy的日志，我们在生产环境中一般单独定义出来。需要将haproxy的info及notice日志分别记录到不同的日志文件中

**

```

vim /etc/haproxy/haproxy.cfg
global
    log /dev/log local0 info
    log /dev/log local0 notice
 
service haproxy restart
 
#需要修改rsyslog配置，为了便于管理。将haproxy相关的配置独立定义到haproxy.conf，并放到/etc/rsyslog.d/下，rsyslog启动时会自动加载此目录下的所有配置文件。
vim /etc/rsyslog.d/haproxy.conf
if ($programname == 'haproxy' and $syslogseverity-text == 'info')
then -/var/log/haproxy/haproxy-info.log
&~
if ($programname == 'haproxy' and $syslogseverity-text == 'notice')
then -/var/log/haproxy/haproxy-notice.log
&~
 
#说明：
这部分配置是将haproxy的info日志记录到/var/log/haproxy/haproxy-info.log下，将notice日志记录到/var/log/haproxy/haproxy-notice.log下。“&~”表示当日志写入到日志文件后，rsyslog停止处理这个信息。
 
systemctl restart rsyslog.service
 
tail -f /var/log/haproxy/haproxy-info.log       #查看haproxy的访问请求日志信息
```

# 八、实例操作：Haproxy集群的日志重新定义

## 1、修改rsyslog配置

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fcf8f4b373a9437182de626ba1a76d4d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

## 2、确认默认haproxy的日志、创建/var/log/haproxy/目录并重启服务

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d00cfa09e679485aa92e54979221940d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e4c8fd08313b4ee18c80290d345bcffc~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f26dd76b448344b0be9fcd0a73e82fce~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

# 九、总结

## 1、对比集群调度工具Haproxy、LVS和Nginx的区别

**区别：**

- LVS基于Linux操作系统内核实现软负载均衡，而HAProxy和Nginx是基于第三方应用实现的软负载均衡
- LVS是可实现4层的IP负载均衡技术，但不支持正则处理，无法实现基于目录、URL的转发。而HAProxy 和Nginx都可以实现4层和7层技术，HAProxy可提供TCP和HTTP应用的负载均衡综合解决方案

- LVS性能最好，但搭建相对复杂，成本较高。一般在100台左右web主机的集群中使用
- nginx做负载均衡调度器，配置简单，管理方便，但并发量不高，且没有主动健康检查。性能没有Haproxy好。可用于并发量不高的场景



## 2、HTTP请求的两种方式

方式： GET、POST方式

区别： GET把参数包含在URL中，POST通过request body传递参数

- GET： 产生一个TCP数据包
- POST： 产生两个TCP数据包

## 3、haproxy配置文件重要参数说明：

**全局配置：**  global **作用：**  用于设定义全局参数，属于进程级的配置，通常与操作系统配置有关。

- maxconn 4096 #进程最大连接数，需考虑"ulimit -n"的限制，推荐使用10240
- daemon #守护进程模式。可以使用非守护进程模式，在生产环境中建议使用守护进程模式
- nbproc 1 #并发进程数，建议与当前服务器CPU核数相等或为其2倍

**默认配置：**  defaults **作用：**  配置默认参数，一般会被应用组件继承。

- retries 3 #检查节点服务器失败次数，连续3次失败，则认为节点不可用
- redispatch #当服务器负载很高时，自动结束当前队列处理比较久的连接
- maxconn 2000 #最大连接数，"defaults"中 的值不能超过“global"段中的定义
- timeout http-request 10s #默认http请求超时时间。建议设置时间为5~10s，增加http连接释放的速度
- timeout http-keep-alive 10s #默认长连接超时时间
- timeout check 10s #设置心跳检查超时时间

**应用组件配置：**  listen **作用：**  一般配置应用模块参数

- option httpchk GET /index. html #检查服务器的index.html文件。发送http的GET请求检查index.html文件，返回2xx、3xx表示正常；返回4xx/5xx表示异常，则隔离该节点。
- balance roundrobin #负载均衡调度算法使用轮询算法roundrobin
- server inst1 192.168.160.40:80 check inter 2000 rise 2 fall 3 #定义在线节点
- server inst2 192.168.160.50:80 check inter 2000 rise 2 fall 3 #定义在线节点

**haproxy支持的最大并发量=并发进程数×每个进程最大连接数，即"nbproc的值 × maxconn的值"**

## 4、日志消息的级别：

| 级号 | 消息    | 级别 | 说明                       |
| ---- | ------- | ---- | -------------------------- |
| 0    | EMERG   | 紧急 | 会导致主机系统不可用的情况 |
| 1    | ALERT   | 警告 | 必须马上采取措施解决的问题 |
| 2    | CRIT    | 严重 | 比较严重的情况             |
| 3    | ERR     | 错误 | 运行出现错误               |
| 4    | WARNING | 提醒 | 可能会影响系统功能的事件   |
| 5    | NOTICE  | 注意 | 不会影响系统但值得注意     |
| 6    | INFO    | 信息 | 一般信息                   |
| 7    | DEBUG   | 调试 | 程序或系统调试信息等       |

## 5、HAProxy负载均衡策略非常多，常见的有如下8种

- **roundrobin∶表示简单的轮询**
- **static-rr∶表示根据权重**
- **leastconn∶ 表示最少连接者先处理**
- source∶ 表示根据请求的源IP，类似Nginx的IP hash机制
- ri∶表示根据请求的URI
- rl_param∶表示根据HTTP请求头来锁定每 一 次HTrTP请求
- rdp-cookie （name）∶表示根据据cookie （name）来锁定并哈希每一次TCP请求

## 6、HAProxy简介

HAProxy是可提供高可用性、负载均衡以及基于TCP和HTTP应用的代理，是免费、快速并且可靠的一种解决方案。HAProxy非常适用于并发大(并发达1w以上)  web站点这些站点通常又需要会话保持或七层处理。HAProxy的运行模式使得它可以很简单安全的整合至当前的架构中，同时可以保护web服务器不被暴露到网络上

## 7、四层代理和七层代理

四层是TCP层，使用IP+端口的方式。类似路由器，只是修改下IP地址，然后转发给后端服务器，TCP三次握手是直接和后端连接的。只不过在后端机器上看到的都是与代理机的IP的established而已
 7层代理则必须要先和代理机三次握手后，才能得到7层（HTT层）的具体内容，然后再转发。意思就是代理机必须要与client和后端的机器都要建立连接。显然性能不行了，但胜在于七层，能写更多的转发规则