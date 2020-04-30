# yum安装JDK
----------------------------
### 卸载自带的JDK
+ 执行命令 rpm -qa | grep java 查看已经安装的jdk
+ 执行命令 rpm -e --nodeps  java-x.x.x-openjdk-xxxxxxx 卸载所有 OpenJDK（有几个卸载几个）
+ 执行完以上步骤后可以再次使用java -version查看是否已经删除成功
### 安装新的JDK
+ 执行命令 yum search java | grep -i --color jdk 查看软件包列表
+ 选择其中的的OpenJDK版本，这里选择JDK1.8的devel版本（带有javac），显示为 java-1.8.0-openjdk-devel.x86_64
+ 执行命令 yum install -y java-1.8.0-openjdk-devel.x86_64 安装JDK
+ 执行命令 java 和 javac 验证安装是否成功
### 配置环境变量
+ yum安装的JDK默认安装路径为：/usr/lib/jvm，转到 /usr/lib/jvm 目录下
+ 可以看到该目录下有一个形如 java-1.8.0-openjdk-1.8.0.xxxxxxxxx.x86_64 的文件夹
+ 执行命令 vim /etc/profile 打开文件并进入编辑模式
+ 在文件末尾追加以下的内容：

       export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.242.b08-1.el7.x86_64
       export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JAVA_HOME/jre/lib/rt.jar
       export PATH=$PATH:$JAVA_HOME/bin

+ 保存并退出
+ 执行命令 source  /etc/profile 使环境变量生效
+ 执行命令 echo $JAVA_HOME 查看是否成功

# yum安装Tomcat
--------------------------------
### 安装Tomcat
+ 执行命令 yum -y install tomcat 安装Tomat7
+ 执行命令 rpm -q tomcat 查看是否安装成功
+ 执行命令 systemctl start tomcat.service 运行 tomcat
+ 执行命令 curl localhost:8080 查看能否访问（这里没有显示，因为容器里面没有项目）
### 开启8080端口（根据情况修改默认端口）
+ 查看防火墙状态，执行命令 systemctl restart firewalld.service
+ 若是处于关闭状态，执行命令 systemctl start firewalld.service 开启防火墙 
+ 执行命令 firewall-cmd --zone=public --add-port=8080/tcp --permanent 开启8080端口，其中：
 
        --zone=public：表示作用域为公共的； 
        --add-port=8080/tcp：添加tcp协议的端口8080；
        --permanent：永久生效，如果没有此参数，则只能维持当前服务生命周期内，重新启动后失效；

+ 执行命令 systemctl restart firewalld.service 重启防火墙
### 配置环境变量
+ 安装的Tomcat默认安装路径为 /usr/share/tomcat
+ 执行命令 vim /etc/prrofile 打开文件并进入编辑模式
+ 在文件末尾追加以下的内容：

       export CATALINA_BASE=/usr/share/tomcat
       export CATALINA_HOME=/usr/share/tomcat
            
+ 保存文件并退出，执行命令 source /etc/profile 使环境变量生效
### 配置Tomcat
+ 执行命令 vim /usr/share/tomcat/conf/tomcat.conf 打开文件并进入编辑模式
+ 在文件末尾加入以下的内容以使tomcat启动8005端口延迟降低:
    
       JAVA_OPTS="-Djava.security.egd=file:/dev/./urandom -Djava.awt.headless=true -Xmx512m -XX:MaxPermSize=256m -XX:+UseConcMarkSweepGC"
        
+ 保存文件并退出	
+ 执行命令 systemctl restart tomcat.service 重新启动Tomcat服务
### 配置管理界面
+ 执行命令 yum install tomcat-webapps tomcat-admin-webapps 安装管理包
+ 执行命令 vim /usr/share/tomcat/conf/tomcat-users.xml 打开文件并进入编辑模式
+ 在<tomcat-users>标签内增加以下内容：
       
       <role rolename="manager-gui"/>
        <role rolename="manager-script" />
        <user username="admin" password="password" roles="manager-gui,manager-script" />

+ 在这里设置用户名为：admin，密码：password
+ 进入 /usr/share/tomcat/webapps/manager/META-INF 文件夹
+ 打开文件 context.xml 并进入编辑模式 
+ 在 <Valve> 标签的属性allow值的末尾增加 |\d+\.\d+\.\d+\.\d+ 表示允许所有主机访问	
+ 执行命令  systemctl restart tomcat.service 重启生效
+ 执行命令 curl localhost:8080 即可看到网页代码
### 配置端口号
+ yum安装的tomcat无法修改8080端口，故需要进行端口映射，在这里将80端口映射到8080
+ 执行命令 iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080
+ 执行命令 service iptables save 保存设置，否则重启后会失效
+ 执行命令 systemctl restart iptables 生效
### 最后
+ 执行命令 systemctl enable tomcat.service 将tomcat服务加入开机启动

# 部署capi
-------------------------------
### 打开tomcat管理界面，输入上面设置的用户名和密码后登陆
### 在要部署的WAR文件一栏中选择capi.war上传，点击部署
+ Tomcat7版本可能过低，部署后无法成功运行，查看日志有如下的出错信息：

	       Caused by: java.lang.ClassNotFoundException: javax.el.ELManager
      	        at org.apache.catalina.loader.WebappClassLoaderBase.loadClass(WebappClassLoaderBase.java:1892)
        	       at org.apache.catalina.loader.WebappClassLoaderBase.loadClass(WebappClassLoaderBase.java:1735)

+ 将提示信息中缺少的类的jar包 javax.el-api-3.0.0.jar 放入tomcat目录下的lib中
+ 执行命令 systemctl restart tomcat.service 重新启动tomcat
+ 将war包重新上传部署，再次访问，项目成功运行

# 进程命令
-------------------------------------
### 查进程
+ ps a 显示现行终端机下的所有程序，包括其他用户的程序。
+ ps -A 显示所有程序。
+ ps -ef|grep java|grep -v grep 显示出所有的java进程，去处掉当前的grep进程
+ ps PID 查看进程的详细信息
### 杀进程
+ kill -9 PID

# 开放及查看端口
----------------------------------
### 开放端口
- firewall-cmd --zone=public --add-port=5672/tcp --permanent   # 开放5672端口
- firewall-cmd --zone=public --remove-port=5672/tcp --permanent  #关闭5672端口
- firewall-cmd --reload   # 配置立即生效
### 查看防火墙所有开放的端口
- firewall-cmd --zone=public --list-ports
### 防火墙的操作
+ systemctl start/restart/stop firewalld
### 查看监听的端口
+ netstat -lnpt
+ netstat -lnpt |grep 5672 检查端口被哪个进程占用=
		