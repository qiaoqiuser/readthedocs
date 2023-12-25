# keepalived的安装与配置

原文：https://blog.csdn.net/qq_40917075/article/details/130578696

keepalived的安装与配置
**1、下载安装包**

下载地址 :keepalived下载地址

http://www.keepalived.org/download.html



**2、把keepalived安装包上传至服务器**

![dba5efad88764aafaba6eef150ead838](img\dba5efad88764aafaba6eef150ead838.png)



**3、解压压缩包**

    tar -zxvf keepalived-2.2.2.tar.gz

![a94b5e3eeb5d434797c55e32c761e8b5](img\a94b5e3eeb5d434797c55e32c761e8b5.png)

![54e977af09d54a90a95c048f30894287](img\54e977af09d54a90a95c048f30894287.png)



**4、安装**

    cd keepalived-2.2.2
    ./configure --prefix=/usr/local/keepalived

![565099625c9543038eede3a0429e0fa0](img\565099625c9543038eede3a0429e0fa0.png)



报错：

![1d0b105c195a4d15801c75c02344fe5d](img\1d0b105c195a4d15801c75c02344fe5d.png)



解决：

    yum -y install libnl libnl-devel

![44136f7b3a114595a015290168a6ed21](img\44136f7b3a114595a015290168a6ed21.png)



再次报错，权限不足,切换root用户，再次执行

![aa4aa638b60b479fb73c297292e4afaf](img\aa4aa638b60b479fb73c297292e4afaf.png)

![cb2d42564e6449f5befff609810e29d1](img\cb2d42564e6449f5befff609810e29d1.png)



完成之后，切回普通用户，再次安装



    su yindy
    ./configure --prefix=/usr/local/keepalived

![3dd52d498a3840f091667fa3609a2bbe](img\3dd52d498a3840f091667fa3609a2bbe.png)



**5、编译验证**

    make && make install 

报错：

![2d37fbcbc2734c32883f263651ad1f6e](img\2d37fbcbc2734c32883f263651ad1f6e.png)



解决：切换用户，再次执行

    su root
    make && make install

![4f4386e9339a454ca35c2fbefd34d9c2](img\4f4386e9339a454ca35c2fbefd34d9c2.png)

![18200edbc56f4cbba19e5f5cfaf35174](img\18200edbc56f4cbba19e5f5cfaf35174.png)



**6、给keepalived相同文件创建链接ln -s ,执行以下命令**

    ln -s /yindy/keepalived-2.2.2/keepalived/etc/init.d/keepalived /etc/rc.d/init.d
    ln -s /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
    ln -s /usr/local/keepalived/sbin/keepalived /usr/sbin/

报错：没有权限，切换root用户,再次执行

    su root 

![e1942babe6594695a522b94d124c803f](img\e1942babe6594695a522b94d124c803f.png)



**7、在/usr/local/keepalived/文件加下创建配置文件**

    cd /usr/local/keepalived/
    touch keepalived.conf

![155695bbaf094c4eaec0cb604d3517e4](img\155695bbaf094c4eaec0cb604d3517e4.png)

![911bb2e4c41b4767a357fb1425de26bd](img\911bb2e4c41b4767a357fb1425de26bd.png)



**8、查看网卡信息**

    ip addr

![bc299d724fae45ddafe4af49b304b386](img\bc299d724fae45ddafe4af49b304b386.png)



**8、编辑keepalived.conf**

master:

	global_defs {
	   router_id LVS_MASTER  #名称标记为master，名字随便取
	   vrrp_gna_interval 0
	}
	
	#加入周期性检测nginx服务脚本的相关配置
	vrrp_script check_nginx{
	    script "/etc/keepalived/check_nginx.sh" #心跳执行的脚本，检测nginx是否启动
	    interval 2                           #（检测脚本执行的间隔，单位是秒）
	}
	
	vrrp_instance VI_1 {
	    state MASTER #指定当前节点为master节点，只能有一个master，其余只能是backup
	    interface ens33 #绑定此虚拟路由使用的网卡的名称，使用ifconfig或者ip addr查看
	    virtual_router_id 99 #指定虚拟路由id，虚拟路由的唯一标识，范围是0-255，mater和backup节点需要指定一样的，相同id为一组
	    priority 200 #指定当前结点的优先级，master节点要大于backup节点。
	    advert_int 1 #指定发送VRRP通告的间隔，单位是秒
	    virtual_ipaddress {
	        192.168.199.130 #指定虚拟ip，自己定义的虚拟ip
	    }
	
	#添加跟踪(执行脚本)
	track_script{
	    check_nginx
	}


backup:

	global_defs {
	   router_id LVS_BACKUP  #名称标记为BACKUP，名字随便取
	   vrrp_gna_interval 0
	}
	
	#加入周期性检测nginx服务脚本的相关配置
	vrrp_script check_nginx{
	    script "/etc/keepalived/check_nginx.sh" #心跳执行的脚本，检测nginx是否启动
	    interval 2                           #（检测脚本执行的间隔，单位是秒）
	}
	
	vrrp_instance VI_1 {
	    state BACKUP #指定当前节点为master节点，只能有一个master，其余只能是backup
	    interface ens33 #绑定此虚拟路由使用的网卡的名称，使用ifconfig或者ip addr查看
	    virtual_router_id 99 #指定虚拟路由id，虚拟路由的唯一标识，范围是0-255，mater和backup节点需要指定一样的，相同id为一组
	    priority 199 #指定当前结点的优先级，master节点要大于backup节点。
	    advert_int 1 #指定发送VRRP通告的间隔，单位是秒
	    virtual_ipaddress {
	        192.168.199.130 #指定虚拟ip，自己定义的虚拟ip
	    }
	
	#添加跟踪(执行脚本)
	track_script{
	    check_nginx
	}


执行命令

    ln -s /usr/local/keepalived/keepalived.conf /etc/keepalived/



**9、编辑监听服务shell脚本check_nginx.sh，并上传到keepalived.conf配置的执行路径/etc/keepalived/**

    #!/bin/bash
    #检测nginx是否启动了
    A=`ps -C nginx --no-header |wc -l`        
    if [ $A -eq 0 ];then    #如果nginx没有启动就启动nginx                        
          /root/ydy/nginx/sbin/nginx                #重启nginx，也可以使直接监听应用服务
          if [ `ps -C nginx --no-header |wc -l` -eq 0 ];then    #nginx重启失败，则停掉keepalived服务，进行VIP转移
                  killall keepalived                    
          fi
    fi



**10、杀掉nginx进程，然后执行脚本，验证脚本正确性**

    ps -ef | grep nginx
    pkill -9 nginx 
    ps -ef | grep nginx
    sh check_nginx.sh
    ps -ef | grep nginx

![95117b178f4d41268a7aeda768f8b495](img\95117b178f4d41268a7aeda768f8b495.png)



**11、查看keepalived的状态**

    service keepalived status
    
    ## systemctl keepalived status

![2b31327ab0c74c5381430b84a5535843](img\2b31327ab0c74c5381430b84a5535843.png)



**12、启动keepalived ,然后查看**

    service keepalived start
    service keepalived status
    ip a

![7056e7b3299c4207872cb8f78128b654](img\7056e7b3299c4207872cb8f78128b654.png)

![4754c6e2b15649d5a479d6b35c6669f0](img\4754c6e2b15649d5a479d6b35c6669f0.png)



**12、验证监听脚本是否可以重启服务**

    ps -ef | grep nginx 
    pkill -9 nginx 
    ps -ef | grep nginx 

![86a22f9d0685482cbf83d6d25aa1874c](img\86a22f9d0685482cbf83d6d25aa1874c.png)



**13、验证keepalived虚拟ip调用**

在浏览器输入http://虚拟ip:port,查看页面是否正常展示。



**14、验证keepalived的高可用**

停掉其中一台服务器的keepalived

    service keepalived stop 
    
    ## systemctl keepalived stop 

再次在浏览器输入http://虚拟ip:port,查看页面是否正常展示。
————————————————
版权声明：本文为CSDN博主「尹家村帅勇」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_40917075/article/details/130578696