# 配置网络
------------------------------
### 开启网络
- 执行命令 ip addr 查看网络配置
- 若没有网络连接，转到 /etc/sysconfig/network-scripts 目录下
- 找到形如 ifcfg-enxxx 文件并打开进入编辑模式
- 将 BOOTPRORO 设置为 static
- 将 ONBOOT 设置为 yes
- 最后，增加以下的内容：
	
	 IPADDR=192.168.83.131     # ip地址
	 NETMASK=255.255.255.0   # 子网掩码
	 GATEWAY=192.168.83.2    # 网关
- 保存并退出，执行命令 system restart network.service 开启网络

# 配置SSH连接
------------------------------
### 安装SSH
- 执行命令 yum list installed | grep openssh-server 查看是否安装了SSH服务
- 若没有，执行命令 yum install openssh-server 安装该服务
- 执行命令 vim /etc/ssh/sshd_config 打开文件并进入编辑模式
- 将以下的配置打开：
	
	Port 22
	ListenAddress 0.0.0.0
	ListenAddress ::
	PermitRootLogin yes
	PasswordAuthentication yes

- 保存并退出
- 执行命令 systemctl restart sshd.service 重新启动SSH

# 安装 JDK1.8.0_251.
-------------------------------
### 卸载系统自带的 OpenJDK
- 执行命令 java -version 查看是否有OpenJDK安装
- 若有，执行命令 rpm -qa | grep java，查看JDK信息
- 执行 rpm -e --nodeps java-x.x.x-openjdk-xxxxxxx 卸载所有 OpenJDK（有几个卸载几个）
### 安装 OracleJDK
- 从Oracle官网下载 JDK8 的安装包 jdk-8u251-linux-x64.tar.gz
-  将文件 jdk-8u251-linux-x64.tar.gz 通过SSH连接工具传入centos7的系统目录 /usr/local 中
- 转到centos7的系统目录 /usr/local 下
- 执行命令 tar -zxvf  jdk-8u251-linux-x64.tar.gz 解压，解压后的文件在当前目录名为 jdk1.8.0_251
### 配置环境变量
- 执行命令 vim /etc/profile 打开文件并进入编辑模式
-  在文件末尾增加下面的几行：

       export JAVA_HOME=/usr/local/jdk1.8.0_251/
       export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
       export PATH=$PATH:$JAVA_HOME/bin
       
- 保存并退出
- 执行命令 source /etc/profile 使环境变量生效
- 执行命令 java -version， 若能看到 java version "1.8.0_251" 字样表示安装成功
- 安装成功后，执行命令 rm -f  jdk-8u251-linux-x64.tar.gz 删除安装包

# 安装Tomcat9.0.34
-------------------------------
### 下载tomcat
-  从tomcat官网下载安装包 apache-tomcat-9.0.34.tar.gz
- 通过SSH工具将安装包传入centos7的系统目录 /usr/local 中
### 解压安装包
- 转到centos7的系统目录 /usr/local 下
- 执行命令 tar -zxvf apache-tomcat-9.0.34.tar.gz ，解压得到 apache-tomcat-9.0.34 文件夹
- 执行命令 mv apache-tomcat-9.0.34  tomcat9.0.34 ，对解压得到的文件夹重命名
### 开启8080端口（根据情况修改默认端口）
- 查看防火墙状态，执行命令 systemctl restart firewalld.service
- 若是处于关闭状态，执行命令 systemctl start firewalld.service 开启防火墙 
- 执行命令 firewall-cmd --zone=public --add-port=8080/tcp --permanent 开启8080端口，其中：
 
        --zone=public：表示作用域为公共的； 
        --add-port=8080/tcp：添加tcp协议的端口8080；
        --permanent：永久生效，如果没有此参数，则只能维持当前服务生命周期内，重新启动后失效；

- 执行命令 systemctl restart firewalld.service 重启防火墙
### 测试tomcat
- 进入/usr/local/tomcat9.0.34/bin 目录下，执行命令  ./startup.sh
- 看到“Tomcat started.”字样后，在外网访问网址：http://服务器IP:8080, 若能看到tomcat主界面，说明tomcat启动成功
- 测试结束后执行命令 ./shutdown.sh 关闭tomcat
### 配置tomcat启动环境
- 执行命令 vi /etc/profile.d/tomcat.sh 并进入编辑模式，写入下面的代码：
	    	
	    export CATALINA_HOME=/usr/local/tomcat9.0.34
		export PATH=$TOMCAT_HOME/bin:$PATH

- 保存并退出，执行 source /etc/profile.d/tomcat.sh 命令使配置生效
- 执行命令 vi /usr/local/tomcat9.0.34/bin/setclasspath.sh 打开文件并进入编辑模式
- 增加下面的代码：

		export JAVA_HOME=/usr/local/jdk1.8.0_251
		export JRE_HOME=/usr/local/jdk1.8.0_251/jre
		
### 配置tomcat开机启动
- 执行命令 vim /usr/lib/systemd/system/tomcat.service，写入以下内容：

		    [unit]
	      	    Description=Tomcat
		    After=syslog.target network.target remote-fs.target nss-lookup.target
		    [Service]
		    Type=forking
		    ExecStart=/usr/local/tomcat9.0.34/bin/startup.sh
		    ExecStop=/usr/local/tomcat9.0.34/bin/shutdown.sh
		    ExecReload=/bin/kill -s HUP $MAINPID
		    ExecStop=/bin/kill -s QUIT $MAINPID
		    RemainAfterExit=yes
		    [Install]
		    WantedBy=multi-user.target
		    
- 保存并退出
- 执行命令 systemctl enable tomcat.service 将tomcat服务设置开机启动
- tomcat服务相关命令如下：

		systemctl enable tomcat.service 随开机启动
		systemctl disabled tomcat.service 禁止开机启动
		systemctl start tomcat.service 启动
		systemctl stop tomcat.service 关闭
		systemctl status tomcat.service 查看
		systemctl restart tomcat.service 重启
		
### 配置tomcat管理界面
- 执行命令 vim /usr/local/tomcat9.0.34/conf/tomcat-users.xml 打开文件并进入编辑模式
- 在<tomcat-users>标签内增加以下内容：
       
       <role rolename="manager-gui"/>
        <role rolename="manager-script" />
        <user username="admin" password="password" roles="manager-gui,manager-script" />

- 在这里设置用户名为：admin，密码：password
- 进入/usr/local/tomcat9.0.34/webapps/manager/META-INF 文件夹
- 打开文件 context.xml 并进入编辑模式 
- 在 <Valve> 标签的属性allow值的末尾增加 |\d+\.\d+\.\d+\.\d+ 表示允许所有主机访问	
- 执行命令  systemctl restart tomcat.service 重启生效
### 配置端口号
- 执行命令 vim /usr/local/tomcat9.0.34/conf/server.xml 打开文件进入编辑模式
- 找到下面的信息

		<Connector port="8080" protocol="HTTP/1.1" 
		 connectionTimeout="20000" redirectPort="8443" />
		 
- 将其中默认的端口8080修改为其他端口，保存并退出
- 执行命令 systemctl restart tomcat.service 重启tomcat使端口生效

# 部署capi
-------------------------------
### 打开tomcat管理界面，输入上面设置的用户名和密码后登陆
### 在要部署的WAR文件一栏中选择capi.war上传，点击部署
		

	          

		
	         
		

	      

	

	
	