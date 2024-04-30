# Jenkins





## 部署 基本配置了改



```bash
#清华源下载导入安装包
#安装JDK
apt update && apt install openjdk-11-jdk
#安装jenkins
dpkg -i jenkins_2.440.1_all.deb
#启动服务默认8080端口
systemctl status jenkins
#初始密码文件位置
cat /var/lib/jenkins/secrets/initialAdminPassword
输入完密码 自定义安装 取消所有勾选的差距，为了初始化提速

#查看启动文件的配置参数 用它自己的用户，指定了数据目录等等
cat /lib/systemd/system/jenkins.service

#重启服务
1. systemctl restart  jenkins
2. http://10.0.0.202:8080/restart
```



### 修改默认密码

![image-20240308131326290](../笔记图片/image-20240308131326290.png)



安装插件，也会安装一些依赖插件，

![image-20240308131524065](../笔记图片/image-20240308131524065.png)





### **小坑**

​    Jenkins默认使用sh  并不是bash



jenkins 命令工具接口

```
wget http://10.0.0.202:8080/jnlpJars/jenkins-cli.jar
复制蓝色连接，下载cli工具
java -jar jenkins-cli.jar -s http://10.0.0.202:8080/ list-plugins
```

![image-20240308132517738](../笔记图片/image-20240308132517738.png)



### ssh 优化

jenkins 要控制其他主机执行任务 基于SSH协议

不论是Key验证 还是密码验证要到手动输入Yes 验证

方法1 依赖Git或Git插件



方法2 

```bash
#修改ssh客户端配置文件，在首次连接别人时，自动输入yes 跳过检查。
vim /etc/ssh/ssh_config
StrictHostKeyChecking no

#对方的公要默认存放到本地家目录
.ssh/known_host
#对方的公钥在他主机的
cat /etc/ssh/
```

![image-20240308133234598](../笔记图片/image-20240308133234598.png)





### 执行器

执行器默认2个，两个任务一起执行，可以修改

![image-20240308133737697](../笔记图片/image-20240308133737697.png)



### jenkins 数据备份 还原 

默认配置文件路径

如果修改，或者不知道，可以在这里查看。

![image-20240308133915634](../笔记图片/image-20240308133915634.png)

#### 数据备份

jenkins 数据备份比较简单，保存数据目录所有文件即可

例如

```bash
tar zcf jenkins-time .tar.gz /var/lib/jenkins/
```



#### 还原 

```bash
#解压把目录中的内容cp到原来数据目录
tar xf jenkins-time .tar.gz 
#把原来数据移走/删除
mv /etc/jenkins/*   /opt/
#把备份数据移动到原来的数据目录
mv etc/jenkins/*   /etc/jenkins/

systemctl restart jenkins
```





### 找回密码

```bash
#停止服务
systemctl stop jenkins
#删除jenkins主目录中Config.xml的部分内容(删除前先备份)
vim /var/lib/jenkins/config.xml
....


....
#重启服务
systemctl restart jenkins
```

![image-20240308134901091](../笔记图片/image-20240308134901091.png)



```bash
删除之后登录jenkins无需安全认证
#在我们登录上去之后，在恢复安全密码认证  jenkins own user database
如果不会没有管理用户选项，无法设置用户密码

#然后设置用户密码
```

![image-20240308135220043](../笔记图片/image-20240308135220043.png)





## CICD流程

### 自由风格

![image-20240308192852053](../笔记图片/image-20240308192852053.png)



Gitlab 和 jenkins 主机做域名解析

```bash
#vim /etc/hosts
10.0.0.202 www.jenkins.jenkins
10.0.0.201 git.git.git
10.0.0.203 生产服务器
```





![image-20240308193241149](../笔记图片/image-20240308193241149.png)



![image-20240308194226140](../笔记图片/image-20240308194226140.png)



### 默认web shell脚本的解释器是SH 建议修改成BASH

他的判断依据是执行到最后是成功还是失败

如果中间错误，不算错误，只是跳过继续往下执行，如果最后执行成功那么就算成功

示例：

  可以尝试输入错误命令，发现并不影响执行

  乱写语法，发现直接就失败了，

![image-20240308194327374](../笔记图片/image-20240308194327374.png)





#### 为了方便我们使用，我们一般是在命令行写脚本，而不是在web shell

注意： 手动编写的脚本一定要放在，jenkins的数据目录下，或者新建目录给目录加上jenkins权限，因为默认是用jenkins用户执行的。



在web中 我们创建项目在项目中 执行操作 

映射到主机上就是  /var/lib/jenkins/项目名/

所以我们执行这个脚本时就在这个目录下，因为在web中的工作目录映射到主机上是同一个目录下，

我的脚本可以不在这个目录里面，因为我写了别的路径调用它

```bash
#新建目录给目录加上jenkins权限
mkdir /data/shell/
chown -R jenkins.jenkins /data/shell
#jenkins的数据目录下
mkdir /var/lib/jenkins/scripts/
********************************************************************
创建好脚本文件后直接在web shell 中写脚本地址调用即可
要记得加执行权限
chmod +x /var/lib/jenkins/scripts/hello.sh
bash -x 调试模式查看命令执行详细过程
```



![image-20240308195958246](../笔记图片/image-20240308195958246.png)







### 实现第一个简单的CICD 前端项目



导入外部项目

默认Gitlab不支持导入，要去管理中心--通用--开启导入设置

![image-20240308202432272](../笔记图片/image-20240308202432272.png)

![image-20240308203832505](../笔记图片/image-20240308203832505.png)



导入项目只之后，我们在201上修改代码然后上传 gitlab

在jenkins上新建项目

然后使用jenkins去gitlab拉取代码，推送到生产服务器

为了方便后续拉取代码方便修改位http  ssh  

![image-20240308203641075](../笔记图片/image-20240308203641075.png)

![image-20240308205233552](../笔记图片/image-20240308205233552.png)





然后使用jenkins去gitlab拉取代码，推送到生产服务器都调用脚本完成

### **用脚本Clone代码，面临的问题 要交互输入用户名密码**

1. 直接把仓库改成公开模式，gitlab要先改群组公开 再修改项目

```bash
#编写脚本
vim /var/lib/jenkins/scripts/http.sh

#克隆源码
git clone http://git.git.git/china/wheel.git
#拷贝到生产
用脚本clone代码会在生成 /var/lib/jenkins/workspace/项目名 

#要考虑连接对方的用户，不写就是以执行脚本的用户连接对方，但对方没有jenkins用户
但是要是已root连接对方要输入yes和密码，因此我们要打通Key验证
scp -r whell/*   root@10.0.0.101:/var/www/html

在jenkins主机上，打通Key验证我们要的执行用户jenkins，所以要生成jenkins用户的公钥拷贝给其他主机
su - jenkins 
jenkins@ubuntu2204:~$ ssh-keygen
jenkins@ubuntu2204:~$ ssh-copy-id root@10.0.0.203

```

 2.解决办法第二头目 下载Gitlab插件，不用把仓库公开，git clone 时直接帮我们验证

不需要在脚本中 git clone 代码

![image-20240308211106272](../笔记图片/image-20240308211106272.png)

![image-20240308211804597](../笔记图片/image-20240308211804597.png)

### 创建凭据

![image-20240308212615469](../笔记图片/image-20240308212615469.png)





修改脚本 jenkins帮我们拉取代码 脚本我们只需要推送代码

```
scp -r *  root@10.0.0.203:/var/www/html
```

![image-20240308213419675](../笔记图片/image-20240308213419675.png)

 3.解决办法第三头目 下载Gitlab插件，不用把仓库公开，git clone 时直接帮我们验证

基于SSH key验证 Clone 代码 

![image-20240308220406693](../笔记图片/image-20240308220406693.png)

传统基于Key验证，是在当前主机生成公私钥，然后把公钥上传给Git服务器

Jenkins现在就相当于是当前主机，要去连接Git服务器，因此他要把自己的公钥上传到Git服务器，他也要有自己的私钥，传统是因为我们的当前主机生成了私钥和公钥，所以只需要上传公钥给Git服务器，现在不仅要上传公钥，还要给jenkins配置私钥，就相当于普通主机默认jenkins没有私钥

```bash
#生成公钥私钥
ssh-keygen
#把公钥上传给Git服务器，他要用这个公钥解密
上传的时候注意，如果你给admin用户添加SSHkey，那么用这个sshkey连接的都有admin权限
cat /root/.ssh/id_rsa.pub
#一组密钥对，jenkins要用这个私钥加密通信，并且认证自己的身份，上传私钥至Jenkins服务器
git 一看我这边有你的公钥  你是我的另一半私钥 咋俩一堆
```



创建凭据，并上传私钥 Key

![image-20240308221652364](../笔记图片/image-20240308221652364.png)



开发修改代码，提交，push到gitlab

我们执行CICD，jenkins 基于ssh连接到gitlab克隆代码到本地，然后执行脚本

登录到某服务器，创建数据目录，把源码CP到数据目录，然后软连接到HTTP服务器的数据目录，方便后续归滚，

```shell
HOST_LIST="
10.0.0.203
"

TIME=`date +%F_%H-%M-%S`

#git clone http://gitlab.wang.org/example/wheel.git


for i in $HOST_LIST;do
   ssh root@$i mkdir -p /opt/wheel-$TIME
   scp -r * root@$i:/opt/wheel-$TIME
   ssh root@$i rm -f /var/www/html
   ssh root@$i ln -s /opt/wheel-$TIME /var/www/html
done
```



#### 脚本回滚

```bash
HOST_LIST="
10.0.0.203
"
APP=wheel
APP_PATH=/var/www/html
DATA_PATH=/opt
DATE=`date +%F_%H-%M-%S`

deploy () {
    for i in ${HOST_LIST};do
        ssh root@$i "rm -f  ${APP_PATH} && mkdir -pv ${DATA_PATH}/${APP}-${DATE}"
        scp -r * root@$i:${DATA_PATH}/${APP}-${DATE}
        ssh root@$i "ln -sv ${DATA_PATH}/${APP}-${DATE} ${APP_PATH}"
    done
}

rollback() {
    for i in ${HOST_LIST};do
        CURRENT_VERISION=$(ssh root@$i "readlink $APP_PATH") 
        CURRENT_VERISION=$(basename ${CURRENT_VERISION})
        echo ${CURRENT_VERISION}
        PRE_VERSION=$(ssh root@$i "ls -1 ${DATA_PATH} | grep -B1 ${CURRENT_VERISION}|head -n1 ")
        echo $PRE_VERSION
        ssh root@$i "rm -f  ${APP_PATH}&& ln -sv ${DATA_PATH}/${PRE_VERSION} ${APP_PATH}"
    done
}


case $1 in 
deploy)
    deploy
    ;;
rollback)
    rollback
    ;;
*)
    exit
    ;;
esac
```



#### 给这个脚本传不同的值 实现部署/回滚

![image-20240308230244814](../笔记图片/image-20240308230244814.png)



### 参数化构建构建任务

给任务设置变量，使用这个变量时传不同的值，然后脚本引用这个变量，就可以通过参数任务参数控制脚本

布尔值参数，真 假 默认为真，取消勾选为假

设置变量的真假值，然后脚本引用这个变量

#### 选项参数

等于菜单，我们可以自定义菜单列表，选择列表中其中一个值，传给变量

![image-20240309092630240](../笔记图片/image-20240309092630240.png)

![image-20240309092722304](../笔记图片/image-20240309092722304.png)

![image-20240309092912980](../笔记图片/image-20240309092912980.png)![image-20240309092913099](../笔记图片/image-20240309092913099.png)





我们有两套分支，要把dev分支部署到测试环境，可以通过选项参数，传入参数

在分支选项中写 变量  然后选项参数控制变量值

![image-20240309094315625](../笔记图片/image-20240309094315625.png)

![image-20240309094351347](../笔记图片/image-20240309094351347.png)

因为这里写什么，jenkins就去仓库Clone什么代码

![image-20240309094501930](../笔记图片/image-20240309094501930.png)





#### 基于TAG部署或回滚

前提是我们得知道有什么TAG

![image-20240309095534498](../笔记图片/image-20240309095534498.png)

![image-20240309095648524](../笔记图片/image-20240309095648524.png)



### Git parameter参数

![image-20240309100615039](../笔记图片/image-20240309100615039.png)



想部署什么版本代码，jenkins就会克隆什么版本代码

![image-20240309100713315](../笔记图片/image-20240309100713315.png)







#### 部署JAVA 项目

Clone源码到我们Gitlab仓库，

jenkins Clone源码他需要编译因此Jenkins服务器上要安装 Maven

```
apt update && apt -y install openjdk-8-jdk && apt  -y install maven
#镜像加速
vim /etc/maven/settings.xml
.....
 <mirror>
	 <id>nexus-aliyun</id>
	 <mirrorOf>*</mirrorOf>
     <name>Nexus aliyun</name>
      <url>http://maven.aliyun.com/nexus/content/groups/public</url>
 </mirror>
 ......
```





#### 配置jenkins拉取此项目的源码

![image-20240309102457943](../笔记图片/image-20240309102457943.png)

#### 编写构建脚本

```
vim /var/lib/jenkins/scripts/hello.sh 

#这里可以IF判断，如果这个目录不存在就创建 存在就往下执行
#mkdir /opt/spring-boot-helloworld/


APP_PATH=/opt/spring-boot-helloworld

HOST_LIST="
10.0.0.203
"
mvn clean package -Dmaven.test.skip=true

for host in $HOST_LIST;do
    ssh root@$host killall -9 java &> /dev/null
    scp target/spring-boot-helloworld-*-SNAPSHOT.jar  root@$host:${APP_PATH}/spring-boot-helloworld.jar
    #ssh root@$host "java -jar ${APP_PATH}/spring-boot-helloworld.jar --server.port=8888 &"
    ssh root@$host "nohup java -jar  ${APP_PATH}/spring-boot-helloworld.jar --server.port=8888  &>/dev/null & "&
done

```

![image-20240309105841573](../笔记图片/image-20240309105841573.png)





![image-20240309105855343](../笔记图片/image-20240309105855343.png)





![image-20240309105902775](../笔记图片/image-20240309105902775.png)





#### TOMCAT war包项目

导入外部War项目  生产机安装 tomcat9

##### Jenkins 新建项目

配置clone源码到本地

![image-20240309111551050](../笔记图片/image-20240309111551050.png)



编写脚本

```shell
APP_PATH=/data/webapps

HOST_LIST="
10.0.0.203
"
mvn clean package -Dmaven.test.skip=true

for host in $HOST_LIST;do
	ssh root@$host rm -rf  ${APP_PATH}/ROOT.war
    ssh root@$host systemctl stop tomcat9
    scp target/hello-world-war-*.war  root@$host:${APP_PATH}/ROOT.war
	ssh root@$host chown -R tomcat.tomcat ${APP_PATH}/
    ssh root@$host systemctl start tomcat9
done
```





### Maven风格任务

准备tomcat服务器 

203机器安装tomcat

依赖于tomcat9 web管理页面

```bash
apt install tomcat9-admin
```



```bash
#默认无法连接tomcat管理页面，需要配置用户授权
vim /etc/tomcat9/tomcat-users.xml
<role rolename="manager-gui"/>     #管理页面要求
<role rolename="manager-script"/>  #插件要求
<user username="tomcat" password="123456" roles="manager-gui,manager-script"/>
[root@ubuntu2204 ~]#systemctl restart tomcat9.service 
```



#### Jenkins 安装 两个插件

**安装 Maven  Integration 插件实现Maven风格的任务** 

**安装 Deploy to container 插件实现连接 tomcat**

![image-20240311193435416](../笔记图片/image-20240311193435416.png)

####  Jenkins 服务器上安装 maven 和配置镜像加速

```
vim /etc/maven/settings.xml
 <mirror>
 <id>nexus-aliyun</id>
 <mirrorOf>*</mirrorOf>
 <name>Nexus aliyun</name>
 <url>http://maven.aliyun.com/nexus/content/groups/public</url>
 </mirror>
 </mirrors>
```



### Jenkins 全局工具配置 JDK (可选)和 Maven(必选)

手动安装 maven  

jenkins要去调用我们的mvn命令编译

```bash
apt update && apt -y install maven
#手动获取mvn安装路径
[root@ubuntu2204 ~]#mvn -v
Apache Maven 3.6.3
Maven home: /usr/share/maven
Java version: 11.0.22, vendor: Ubuntu, runtime: /usr/lib/jvm/java-11-openjdk-amd64
Default locale: zh_CN, platform encoding: UTF-8
OS name: "linux", version: "5.15.0-100-generic", arch: "amd64", family: "unix"
```

![image-20240311193907745](../笔记图片/image-20240311193907745.png)



![image-20240311194221099](C:\Users\50569\Desktop\运维笔记\笔记图片\image-20240311194221099.png)



![image-20240430180436171](../笔记图片/image-20240430180436171.png)



### 创建 Tomcat 的全局凭据  

根据tomcat的用户权限配置,创建jenkins连接tomcat的用户和权限，

也就是jenkins要连接到tomcat的web管理界面

![image-20240311194744798](../笔记图片/image-20240311194744798.png)



### 创建maven任务

![image-20240311194837664](../笔记图片/image-20240311194837664.png)



#### 源码地址

![image-20240311194917243](../笔记图片/image-20240311194917243.png)

#### 配置编译时的选项

![image-20240311195325081](../笔记图片/image-20240311195325081.png)

![image-20240311195744501](../笔记图片/image-20240311195744501.png)

#### 构建后操作  也即是编译完后我们要把文件cp到生产主机 运行

![image-20240311195925599](../笔记图片/image-20240311195925599.png)

![image-20240311195941362](../笔记图片/image-20240311195941362.png)



![image-20240311200240917](../笔记图片/image-20240311200240917.png)







### Golang项目

#### 在jenkins服务器上安装golang编译环境

因为要在本地编译好二进制包，分发到生产环境

```bash
apt update && apt -y install golang
go version
```

#### Gitlab导入互联网测试项目

```
https://gitee.com/lbtooth/ginweb.git
```

#### 准备环境

204安装mysql redis   因为此项目要使用数据库

```bash
apt update && apt install redis mysql-server

#查看geweb项目 ini文件按照此文件匹配环境
#把源码克隆到本地修改配置，然后上传到gitlab
git clone git@git.git.git:china/ginweb.git

#修改go项目的MySQL和Redis的配置
vim ginweb/conf/ginweb.ini

[mysql]
host = "10.0.0.204"
port = 3306
databases = "ginweb"
user = "ginweb"
passwd = "123456"

[redis]
host = "10.0.0.204"
port = 6379
passwd = "123456"

git commit -am v2
git push

#mysql修改配置，并创建对应的账户密码数据库，准备表结构（克隆源码，源码中自带脚本文件）
sed -i '/127.0.0.1/s/^/#/'  /etc/mysql/mysql.conf.d/mysqld.cnf
systemctl restart mysql.service
#创建库和用户
create database ginweb;
create user ginweb@'10.0.0.%' identified by '123456';
grant all on ginweb.* to ginweb@'10.0.0.%';
#导入项目自带的表结构
mysql -uginweb -p123456 -h10.0.0.204 ginweb  < ginweb.sql


#修改redis配置
vim /etc/redis/redis.conf
bind 0.0.0.0
requirepass 123456

systemctl restart redis


#此项目默认使用8888端口
#关闭占用8888/tcp的进程防止冲突
fuser -k 8888/tcp
```

203跑web服务



#### 编写shell 脚本



```shell
vim /var/lib/jenkins/scripts/go.sh

APP=ginweb
APP_PATH=/data
DATE=`date +%F_%H-%M-%S`
HOST_LIST="
10.0.0.203"

build () {
	#go env 可以查看到下面变量信息，如下环境变量不支持相对路径，只支持绝对路径
	#root用户运行脚本
	#export GOCACHE="/root/.cache/go-build"
	#export GOPATH="/root/go"
	#Jenkins用户运行脚本
	export GOCACHE="/var/lib/jenkins/.cache/go-build"
	export GOPATH="/var/lib/jenkins/go"
	#go env -w GOPROXY=https://goproxy.cn,direct
	export GOPROXY="https://goproxy.cn,direct"
	CGO_ENABLED=0 go build -o ${APP}
}

 deloy () {
	for host in $HOST_LIST;do
		ssh root@$host "mkdir -p $APP_PATH/${APP}-${DATE}"
        scp -r *  root@$host:$APP_PATH/${APP}-${DATE}/
		ssh root@$host "killall -0 ${APP} &> /dev/null  && killall -9 ${APP}; rm -f ${APP_PATH}/${APP} && \
		ln -s ${APP_PATH}/${APP}-${DATE} ${APP_PATH}/${APP}; \
		cd ${APP_PATH}/${APP}/ && nohup ./${APP}&>/dev/null" &
 done
}
build

deloy


```

#### 如果Jenkins bash -x /var/lib/jenkins/scripts/go.sh  就不用加执行权限



### 创建jenkins项目

![image-20240311205254519](../笔记图片/image-20240311205254519.png)



![image-20240311205346948](../笔记图片/image-20240311205346948.png)



#### 调用脚本

![image-20240311205510840](../笔记图片/image-20240311205510840.png)









### Ansible

![image-20240311210403514](../笔记图片/image-20240311210403514.png)

#### jenkins 安装 Ansible 环境

```bash
apt update && apt -y install ansible 
#Ubuntu22.04默认没有配置文件，可以手动创建一个空文件，使用默认值即可
#ansible --version
mkdir -p /etc/ansible/ && touch /etc/ansible/ansible.cfg

#准备主机清单文件
默认以运行ansible用户连接对方，但是对方主机没有jenkins用户，所以我们要写死要以root连接
[root@jenkins ~]#vim /etc/ansible/hosts
[web]
10.0.0.202  ansible_ssh_user=root
[web1]
10.0.0.203  ansible_ssh_user=root

#因为Jenkins服务是以jenkins用户身份运行，所以需要实现Jenkins用户到被控制端的免密码验证
su - jenkins
ssh-copy-id root@10.0.0.203
#测试是否打通
su - jenkins
ansible all -u root -m ping

```



#### jenkins web安装 Ansible 插件

![image-20240311211334187](../笔记图片/image-20240311211334187.png)

##### **安装完增加了一些新的选项**

![image-20240311211544499](../笔记图片/image-20240311211544499.png)

#### 使用 Ansible Ad-Hoc 实现任务

![image-20240311212140447](../笔记图片/image-20240311212140447.png)

#### 使用 Ansible Playbook 实现任务



为了复用一个playbook实现在生产环境和测试环境的部署

我们可以写两个不同的 hosts文件，在jenkins中调用ansible时选择不同的hosts文件从而实现不同环境的部署，而且playbook文件不用修改。

注意文件不同但是文件中的主机列表名称一样，但主机列表中的主机IP不一样

示例：

这样我们只需要选择使用不同的hosts文件，实现不同环境的部署，而不用修改playbook,相同的主机组名对应不同的IP地址

```bash
#生产环境 hosts 文件
vim /etc/ansible/produce_hosts
[web]
10.0.0.200   ansibles_ssh_user=root
```

```bash
#测试环境 hosts 文件
vim /etc/ansible/test_hosts
[web]
10.0.0.100	 ansibles_ssh_user=root
```

```yaml
#playbook文件
- hosts: web
  remote_user: root
  
  tasks:
  -name: cmd ....
   shell: 
   	 cmd: hostname
```

#### 构建参数化任务

通过参数化任务 设定一个变量 在运行此任务时给这个变量赋值，而这个变量的值就是我们hosts文件的路径，然后在ansible任务选项里 写上我们的变量，从而实现动态修改部署环境。

简单说，我们运行任务时候点击选项这个选项的值会传到ansible指定hosts文件的文本框

**定义参数化构建变量 这个变量的值 就是我们不同hosts文件的路径**

![image-20240311213314702](../笔记图片/image-20240311213314702.png)

playbook文件用同一个，但是主机文件是通过我们变量的值选择，选择不同的文件从而部署到不同的环境

![image-20240311213540692](../笔记图片/image-20240311213540692.png)

![image-20240311213641142](../笔记图片/image-20240311213641142.png)







#### 完美解决方案

假如我们有两套生产环境和测试环境，在生产环境的hosts文件中，我们可能有多组主机列表，而各个主机组的名称不一样。

但如果按照上面写死hosts列表，hosts文件中有多组主机列表，那如果要在其他生产环境部署，还是需要修改

Playbook的hosts字段，因此要hosts字段要写成变量，在运行playbook时，给我们定义好的变量赋值，从而实现在不同环境下的多组主机的灵活部署，而不用修改playbook

如果写死hosts字段，部署到同一个hosts文件的另一组主机，还是要修改

示例：

```bash
#生产环境 hosts 文件
vim /etc/ansible/produce_hosts
[web]
10.0.0.200   ansibles_ssh_user=root
10.0.0.201
[web2]
10.0.0.210   
10.0.0.211
```



```bash
#测试环境 hosts 文件
vim /etc/ansible/test_hosts
[web]
10.0.0.100	 ansibles_ssh_user=root
10.0.0.101
[web2]
10.0.0.110	 ansibles_ssh_user=root
10.0.0.111
```



```yml
#playbook文件
- hosts: “{{ server }}”
  remote_user: root
  
  tasks:
  -name: cmd ....
   shell: 
   	 cmd:
```

#### 构建参数化任务

先在Jenkins中创建变量赋值，因为jenkins和ansible的变量不是一回事，然后再创建ansible变量调用jenkins变量的值

**增加选项 此参数选项用来觉得主机组**

![image-20240311214427945](../笔记图片/image-20240311214427945.png)



但是这值这个变量是jenkins中的，不是playbook中的变量

我们要把jenkins变量的值传给playbook中的变量

![image-20240311214633108](../笔记图片/image-20240311214633108.png)



${group}

![image-20240311215009269](../笔记图片/image-20240311215009269.png)



上面的选项用来决定用那个hosts文件，也就是那套主机组文件

下面的选项用来决定，hosts文件中的那组主机

![image-20240311215059237](../笔记图片/image-20240311215059237.png)



### 构建后通知

#### 准备告警邮箱配置

生成邮箱登录授权码，可以使用QQ或163邮箱等

![image-20240311222613960](../笔记图片/image-20240311222613960.png)

```bash
#QQ邮箱授权码
efinhpqmhmerbigi
smtp.qq.com

smtp.163.com

成功不发
失败发
失败转换为成功发
```

系统设置中

![image-20240311222803177](../笔记图片/image-20240311222803177.png)

![image-20240311223151374](../笔记图片/image-20240311223151374.png)

![image-20240311223511063](../笔记图片/image-20240311223511063.png)



#### 在项目项目中调用

构建后操作

![image-20240311223707973](../笔记图片/image-20240311223707973.png)





![image-20240311223734706](../笔记图片/image-20240311223734706.png)

#### ding talk

创建dd群聊

![image-20240311224816959](../笔记图片/image-20240311224816959.png)

![image-20240311224922324](../笔记图片/image-20240311224922324.png)



把机器人的webhook地址复制走

##### 在jenkins 中安装插件  Ding talk 安装完重启服务



![image-20240311225318993](../笔记图片/image-20240311225318993.png)

![image-20240311225430857](../笔记图片/image-20240311225430857.png)

![image-20240311225517183](../笔记图片/image-20240311225517183.png)





#### 企业微信通知

1创建群聊，在群聊中添加机器人，保存机器人的信息

```
https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=93e36d46-5f57-45a5-9871-954ea8645f4d
```

Jenkins安装[Qy Wechat Notification](https://plugins.jenkins.io/qy-wechat-notification)插件

![image-20240311230050435](../笔记图片/image-20240311230050435.png)





![image-20240311230420443](../笔记图片/image-20240311230420443.png)





![image-20240311230606963](../笔记图片/image-20240311230606963.png)





自定义格式信息

```
- 构建ID: ${BUILD_ID}
- 部署项目: ${JOB_NAME}
- 部署目录: ${WORKSPACE}
```





### 自动构建

H的含义

按照传统的定时任务

1 1 * * *  每天的 1点的01分执行任务，如果有很多任务都用的是这个时间点，大量的任务都在这个时间点执行，容易影响jenkins性能

H/10  1 * * *  每天的1点 十分钟执行一次，H的意思就是防止大量任务都在同一个十分执行，正常时从0分钟开始到10分执行，H/10就是可能时 1分钟的十分钟，也能是从15分钟开始间隔10钟执行，把任务错开执行。

H（0-29）/10  * * * * 每天每小时每分钟每隔十分钟执行一次，但他可能是从 0-29分钟中的任意时间点开始间隔10分钟执行一次，

H * * * *  每小时 任意内分钟执行一次

#### 传统的周期性构建

到时间就触发构建，不是很人性化，如果我们的代码没变，没更新，它到时间旧构建，根本不管你代码是否更新

![image-20240312093218243](../笔记图片/image-20240312093218243.png)



#### 轮询CSM

定期到代码仓库看看代码有无变化，代码更新了才构建

![image-20240312093301484](../笔记图片/image-20240312093301484.png)

不论是定时还是SCM 都是 jenkins主动到 gitlab上拉代码或者询问拉代码

![image-20240312094103253](../笔记图片/image-20240312094103253.png)

#### 触发远程构建（自带功能）

![image-20240312094630071](../笔记图片/image-20240312094630071.png)



```bash
#按照官方示例修改连接
JENKINS_URL/job/webhook1/build?token=TOKEN_NAME 
http://10.0.0.202:8080/job/webhook1/build?token=123456

#但是gitlab结合时，还需要加用户和Token验证（不再支持密码）才能实现
http://user:<token>@10.0.0.202:8080/job/webhook1/build?token=123456


#如果执行正常，则无任何显示
token：11f6d6e469400dbae1b7561d3b183d4166
#测试
curl http://admin:11f6d6e469400dbae1b7561d3b183d4166@10.0.0.202:8080/job/webhook1/build?token=123456
```

在对应的用户上生成API token，gitlab要连接我们的jenkins网站，要输入账号密码登录，然后再用任务token认证，但是账号密码登录的方式被取消了（不安全），更换为账号和token，这就意味我们要再对应的账号生成token

gitlab----admin@<token>登录到jenkins-----验证任务token----触发执行



生成token 只能生成的时候保存，不用的时候可以删除token

![image-20240312095418930](../笔记图片/image-20240312095418930.png)

![image-20240312095450321](../笔记图片/image-20240312095450321.png)

#### 再gitlab上配置主动触发

##### gitlab 默认不允许任何请求出战，需要修改配置

管理中心----设置-----网络

![image-20240312100544769](../笔记图片/image-20240312100544769.png)





##### 想要自动触发那个项目就再gitlab中的那个项目中设置，而且项目要对应jenkins中的

![image-20240312100158111](../笔记图片/image-20240312100158111.png)

![image-20240312100258874](../笔记图片/image-20240312100258874.png)



![image-20240312100419918](../笔记图片/image-20240312100419918.png)



#### 通过插件实现



```
token=22eb2e37eb9063083e5acf5f02ed6efd

http://10.0.0.202:8080/project/webhook1
```

安装gitlab插件自动安装的插件 

![image-20240312101741959](../笔记图片/image-20240312101741959.png)



##### 生成token

![image-20240312101853512](../笔记图片/image-20240312101853512.png)





Gitlab设置

![image-20240312102033802](../笔记图片/image-20240312102033802.png)



![image-20240312102146869](../笔记图片/image-20240312102146869.png)



![image-20240312102218066](../笔记图片/image-20240312102218066.png)



### 构建前后关联其他项目

在前面任务中利用构建后操作关联后续任务 

在后面任务中利用构建触发器关联前面任务

![image-20240311215745509](../笔记图片/image-20240311215745509.png)

创建三个job简单任务

![image-20240311220041368](../笔记图片/image-20240311220041368.png)



在job1中添加构建后动作，job2 逗号隔开可以写多个

![image-20240311220422102](../笔记图片/image-20240311220422102.png)





在job3上设置 利用构建触发器关联前面任务

![image-20240311220719264](../笔记图片/image-20240311220719264.png)



### 实现容器化Docker任务

![image-20240312102943293](../笔记图片/image-20240312102943293.png)



#### 基础环境准备

部署harbor

```bash
apt update  &&  apt install docker-compose
#导入harbor安装包
mkdir /apps
tar -xf harbor-offline-installer-v2.10.0.tgz  -C /apps/
#创建配置文件
cp harbor.yml.tmpl harbor.yml

vim  harbor.yml
hostname: www.zed.io
#注释掉 https的配置
harbor_admin_password = 123456 #修改此行指定harbor登录用户admin的密码

install.sh
```

**在harbor创建项目使用的账户**

![image-20240312113342608](../笔记图片/image-20240312113342608.png)

**创建项目exampl使用的仓库，并且把zedking用户**加入此项目

![image-20240312113653382](../笔记图片/image-20240312113653382.png)



![image-20240312113740043](../笔记图片/image-20240312113740043.png)













##### jenkins环境初始

在jenkins主机安装 Docker，maven，并且信任harbor，配置harbor的域名解析

```bash
#jenkins要打包、打镜像、上传镜像，要连接客户端
apt update && apt -y install docker.io maven

#域名解析
vim /etc/hosts
10.0.0.204 www.zed.io

#信任harbor
vim /lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --insecure-registry www.zed.io
#重载配置/重启服务
systemctl daemon-reload 
systemctl restart docker.service

#jenkins用户要使用docker.socker文件和dockerDeamen 通信，默认docker.socker只允许docker组 用户访问，因此我们要把jenkins用户添加到docker组中
usermod -aG docker jenkins
#我们启动jenkins时，jenkins用户不在docker组中，在之后我们添加到docker组中，新的权限不生效，所以要重启jenkins
systemctl restart jenkins

#jenkins要上传镜像还要登录和harbor
注意：是jenkinsy用户要登录harbor上传镜像，所以要切换jenkins用户 login harbor，如果用root login 那么认证信息就保存到/root/.docker/config.json，而不再jenkins用户的加目录下，导致认证失败。
su - jenkins
docker login www.zed.io
```







#### 生产机器配置

在目标主机安装 Docker，并且打开远程连接端口，并且信任harbor，配置harbor的域名解析因为他要去harbor下载镜像

```bash
#在生成203机器上，安装docker运行环境，并且打开远程端口，因为jenkins要连接过来执行命令，后续要拉取镜像，因此要信任harbor
apt update && apt -y install docker.io

#域名解析
vim /etc/hosts
10.0.0.204 www.zed.io

#信任harbor 并开启远程连接
vim /lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --insecure-registry www.zed.io -H tcp://0.0.0.0:2375
#重载配置/重启服务
systemctl daemon-reload 
systemctl restart docker.service
```



















#### jenkins实现脚本自由风格任务

![image-20240312115349563](../笔记图片/image-20240312115349563.png)

![image-20240312115438431](../笔记图片/image-20240312115438431.png)

![image-20240312115535320](../笔记图片/image-20240312115535320.png)





**编写脚本**

```bash
vim /var/lib/jenkins/scripts/docker.sh
REGISTRY=www.zed.io
PORT=8888
HOSTS="
10.0.0.203
"
mvn clean package -Dmaven.test.skip=true

docker build -t ${REGISTRY}/example/myapp:v$BUILD_ID .
docker push ${REGISTRY}/example/myapp:v$BUILD_ID

for i in $HOSTS;do
	docker -H $i rm -f myapp
	docker -H $i run -d  -p ${PORT}:8888 --name myapp ${REGISTRY}/example/myapp:v$BUILD_ID
	#ssh root@$i docker rm -f myapp
	#ssh root@$i docker run  -d  -p ${PORT}:8888 --name myapp ${REGISTRY}/example/myapp:v$BUILD_ID
done
```



### 使用Docker插件 完成任务构建

Docker插件===Dcoker-client

Dcoker-client要通过socker文件和Dockerd通讯，执行构建操作，默认docker，sock文件只允许docker组内的用户连接，要把jenkins 加入docker组里面，重启jenkins

基于端口连接方式 tcp

本地sock文件   unix://



```bash
#这种配置方法需要，让jenkins通过插件连接的docker服务器开启远程连接
这两种方法要自己尝试支持那种，不同的版本可能支持一样
tcp://10.0.0.202:2375
tcp://localhost:2375
unix://localhost:2375

#如果Jenkins以Jenkins 用户运行，需要将Jenkins用户添加到docker组中,否则默认jenkins用户没有权限 使用docker对应的socket文件
unix:///var/run/docker.sock
usermod -aG docker jenkins
id jenkins
#需要重启jenkins上面的权限才能生效
systemctl restart jenkins
```



####  安装插件 docker-build-step

![image-20240313125128424](../笔记图片/image-20240313125128424.png)



#### 在 Jenkins 创建连接 Harbor 的凭证

插件要作为docker客户端连接到Harbor上传镜像

![image-20240313131305148](../笔记图片/image-20240313131305148.png)



#### 创建项目 执行第一个构建动作

因为插件无法源码编译

![image-20240313131911698](../笔记图片/image-20240313131911698.png)

#### 执行第二个构建动作、

构建镜像

![image-20240313132531269](../笔记图片/image-20240313132531269.png)

![image-20240313135025371](../笔记图片/image-20240313135025371.png)

#### **执行第三个构建动作**

上传镜像

![image-20240313132851898](../笔记图片/image-20240313132851898.png)

#### 执行第四个动作

启动服务

编写脚本

```shell
#!/bin/bash
list="
10.0.0.203
"
 for  i in $list ;do
 #ssh root@$i docker rm -f $JOB_NAME
 #ssh root@$i docker run  -d  -p 80:8888 --name $JOB_NAME harbor.wang.org/example/$JOB_NAME:$BUILD_NUMBER
     docker -H $i rm  -f $JOB_NAME  || true
     docker -H $i run -d -p 80:8888 --name $JOB_NAME  
www.zed.io/exampl/$JOB_NAME:$BUILD_NUMBER
done
```



```
#!/bin/bash
 for  i in {01..02};do
 #ssh root@$i docker rm -f $JOB_NAME
 #ssh root@$i docker run  -d  -p 80:8888 --name $JOB_NAME 
harbor.wang.org/example/$JOB_NAME:$BUILD_NUMBER
     docker -H web${i}.wang.org:2375 rm  -f $JOB_NAME  || true
     docker -H web${i}.wang.org:2375 run -d -p 80:8888 --name $JOB_NAME  
harbor.wang.org/example/$JOB_NAME:$BUILD_NUMBER
 done
```





## Jenkins高级功能

#### jenkins 分布式



#### Jenkins Master 与 Agent之间的通信方式

![image-20240313195616323](../笔记图片/image-20240313195616323.png)

#### Launch agent via SSH

此方式需要安装SSH Build Agents插件

Master通过SSH协议连接 agent



#### Launch agent by connecting it to the controller

此方式中文翻译为通过 Java Web 启动代理

要求：Controller端额外提供一个套接字以接收连接请求，默认使用tcp协议的50000端口，也支持 使用随机端口（安全，可能会对服务端在防火墙开放该端口造成困扰），也可以使用websocket， 基于默认8080端口建立集群通信连接



### Agent 分类

静态Agent：

固定的持续运行的Agent,即使没有任务,也需要启动Agent 

动态Agent：

容器化环境

按需动态创建和删除 Agent ,当无任务执行时,删除Agent





### 实战案例: 基于 SSH 协议实现 Jenkins 分布式（静态）



![image-20240313200228836](../笔记图片/image-20240313200228836.png)

####  Slave 节点安装 Java 等环境确保和 Master 环境一致

Slave 节点通过从Master节点自动下载的基于 JAVA 的 remoting.jar 程序包实现,所以需要安装JDK

Slave服务器需要创建与Master相同的数据目录，因为脚本中调用的路径只有相对于Master的一个路 径，此路径在master与各node节点应该保持一致。任务中执行的脚本存放的路径和master也必须一致.

如果Slave需要执行编译或执行特定的job，则也需要配置Java或其它语言环境,安装 git、maven、go、 ansible等与master相同的基础运行环境

agent要去代码仓库拉代码所以要做域名解析，不然找不到代码仓库

docker容器跑agent 要注意：第一次连接GIT拉去代码SSH协议，要输入Yes，所以要登录到容器输入yes，宿主机要配置 ssh_config  跳过sshe yes/no

#### **全局安全配置中配置 优化**

![image-20240313221355322](../笔记图片/image-20240313221355322.png)

**注意： Jenkins Agent 和 Master 的环境尽可能一致，包括软件的版本，路径，脚本，ssh key验证等**



```bash
#安装和Master节点相同版本的jdk
apt update && apt install -y openjdk-11-jdk
#名称解析要和jenkins服务器一致

#生成ssh key,并复制公钥到Gitlab的相关联的用户
jenkins是客户端，连接agent是服务端
客户端连接服务端先把自己的公钥上传到客户端，然后用自己的私钥加密，服务端收到后用 公钥解密
ssh-keygen
ssh-copy-id 
```



#### Master 节点安装插件

![image-20240313200706033](../笔记图片/image-20240313200706033.png)





#### 新建节点

![image-20240313200916479](../笔记图片/image-20240313200916479.png)



![image-20240313201012017](../笔记图片/image-20240313201012017.png)



#### 添加 Master 访问 Slave 认证凭据

Master要通过SSH连接登录到slave并执行任务

master是客户端，slave是服务端，把master的公钥上传至服务器 

账号密码无所谓直接填写就OK

![image-20240313201739338](../笔记图片/image-20240313201739338.png)

![image-20240313201930694](../笔记图片/image-20240313201930694.png)

![image-20240313202318110](../笔记图片/image-20240313202318110.png)

![image-20240313202424320](../笔记图片/image-20240313202424320.png)

#### 如果使用脚本，那么要保证agent的工作目录也有这个脚本，而且如果脚本要连接其他主机，agent也要做其他主机的key验证 包括跳过sshd  yes

![image-20240313205505916](../笔记图片/image-20240313205505916.png)





### 实战案例: 基于docker-compose 通过SSH 实现 Jenkins 分布 式

```yaml
#agent节点安装
apt install docker-composem

vim docker-compose.yaml
 version: '3.6'
 volumes:
  ssh_agent01_data: {}
 networks:
  jenkins_net:
    driver: bridge
    ipam:
      config:- subnet: 172.27.0.0/24
 services:
  ssh-agent01:
    image: jenkins/ssh-agent:jdk11
    hostname: ssh-agent01.wang.org
 #user: jenkins
    environment:
      TZ: Asia/Shanghai
 #JENKINS_AGENT_HOME: /home/jenkins
      JENKINS_AGENT_SSH_PUBKEY: 
   这里的公钥，jenkins要作为客户端连接agent，jenkins要创建凭据把自己的私钥上传，然后这里要填写私钥对应的公钥
 # SSH PRIVATE KEY and PUBLIC KEY 需要手动生成,利用private key 创建凭据,此处写入 public key
    networks:
      jenkins_net:
        ipv4_address: 172.27.0.21
        aliases:- ssh-slave01- ssh-agent01
    ports:- "22022:22"
 #restart: always
```





#### 在 Jenkins master 创建 Agent

![image-20240313214556200](../笔记图片/image-20240313214556200.png)

###  





### 实战案例：基于 JNLP协议的 Java Web 启动代理

Launch agent by connecting it to the controller 也称为 通过 Java Web 启动代理  

此方式无需安装插件，即可实现

**全局安全配置**

因为agetn要连接我们，所以我们要先打开端口，再启动agent，不然agent起来也无法连接

![image-20240313223028026](../笔记图片/image-20240313223028026.png)

#### 在jenkins上创建agent

![image-20240313224009972](../笔记图片/image-20240313224009972.png)

#### 在agent节点执行

要做域名解析

安装jdk

![image-20240313224249108](../笔记图片/image-20240313224249108.png)





### 启动 Docker Compose 容器的 Jenkins Agent

#### 要先在jenkins上创建主机







```bash
version: '3.6'
 volumes:
  agent01_data: {}
  slave02_data: {}
 networks:
  jenkins_net:
    driver: bridge
    ipam:
      config:
      - subnet: 172.27.0.0/24
 services:
  agent01:
    image: jenkins/inbound-agent:alpine-jdk11
    hostname: agent01.wang.org
    user: root
    environment:
      TZ: Asia/Shanghai
      JENKINS_URL: http://jenkins.wang.org:8080 #指定jenkins服务器地址
      JENKINS_AGENT_NAME: jenkins-jnlp-agent01  #指定和Jenkins的Agent的名称相同
      JENKINS_AGENT_WORKDIR: /data/jenkins/
      JENKINS_SECRET: 3aef4f69d569d9f3b48e2daf91c3a80b1c8d8bcb310462859950dacc1b202cbc
#此处的JENKINS_SECRET需要用创建agent时生成的Secret替代
    volumes:- agent01_data:/data/jenkins/
    networks:
      jenkins_net:
        ipv4_address: 172.27.0.11
        aliases:
        - slave01
        - agent01
```





###  实战案例：基于Docker的动态Agent 

205安装docker，因为jenkins模拟客户端远程连接 205 docker engine 执行容器构建，所以要开启远程登录

```bash
apt update && apt -y install docker.io
#修改配置
vim /lib/systemd/system/docker.service
[Service]
Type=notify
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock  -H tcp://0.0.0.0:2375
#重启服务
systemctl daemon-reload && systemctl restart docker.service
```



#### 安装 Docker 插件

![image-20240314091147437](../笔记图片/image-20240314091147437.png)

#### 创建云

![image-20240314091440663](../笔记图片/image-20240314091440663.png)

![image-20240314091457109](../笔记图片/image-20240314091457109.png)

![image-20240314091557122](../笔记图片/image-20240314091557122.png)

#### Docker Cloud Details 配置指定连接Docker的方式

方式1: 本地 socket文件

此方式表示Agent 对应的容器也是运行在 Jenkins Master 主机上，不是真正意义的Jenkins分布式

```bash
#本地路径
unix:///var/run/docker.sock

ll /var/run/docker.sock 
usermod -aG docker jenkins && systemctl restart jenkins
```

 方式2: 远程方式

jenkins模拟docker客户端连接 205的docker 服务端，jenkins其实不用安装docker

![image-20240314092048147](../笔记图片/image-20240314092048147.png)

#### 添加 Docker Agent templates

```
#可以在Docker主机提前拉取镜像
docker pull jenkins/inbound-agent:jdk11        
docker pull jenkins/inbound-agent:alpine-jdk11 #推荐

http://www.jenkins.jenkins:8080

10.0.0.202 www.jenkins.jenkins
10.0.0.201 git.git.git
```

#### JNLP

![image-20240314092837302](../笔记图片/image-20240314092837302.png)

![image-20240314093312290](../笔记图片/image-20240314093312290.png)

#### 注意：如果基于JNLP协议，必须要实现Jenkins主机的名称解析，如下显示

JNLP协议需要Docker 主机主动连接 Jenkins主机，所以需要确保Docker Engine 主机可以解析 www.jenkins.jenkins 的名称

而SSH协议是 Jenkins 主机主动连接 Docker 主机，所以无需在容器内注入名称解析

![image-20240314093612564](../笔记图片/image-20240314093612564.png)

**要添加到容器的 /etc/hosts 文件中的新行分隔主机名/IP 映射的列表。以“hostname：IP”格式指定。**

![image-20240314093735867](../笔记图片/image-20240314093735867.png)



#### SSH

```bash
#镜像
jenkins/ssh-agent:jdk11
```

添加模板

![image-20240314094819758](../笔记图片/image-20240314094819758.png)

![image-20240314095121196](../笔记图片/image-20240314095121196.png)

#### 注入域名解析

容器无法解析git仓库地址

![image-20240314095854282](../笔记图片/image-20240314095854282.png)



#### 创建自由风格任务

先用标签筛选出用那个 agent主机执行任务

然后选择构建任务，在这个机器上执行其他构建操作，可以调用插件等等

agent容器，编译--构建镜像---上传harbor

任务 buil step== 拉取这个镜像 跑起来



![image-20240314100457963](../笔记图片/image-20240314100457963.png)

![image-20240314100837409](../笔记图片/image-20240314100837409.png)



### Pipeline

#### 安装 Pipeline 和 Pipeline Stage View 插件

![image-20240315193522104](../笔记图片/image-20240315193522104.png)

#### 构建推送镜像Pipeline

采用凭据harbor登录

里面要用的凭据要提前创建好



```
   stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'harbor-passwd', \
                        passwordVariable: 'harborPassword', usernameVariable: 'harborUserName')]) 
 从凭据取出账号密码，赋值给变量，可以提高安全性
```



```
pipeline {
    agent any
    tools {
        maven 'Apache Maven 3.6.3'
    }
    environment {
        codeRepo="git@git.git.git:china/hello.git"
        credential="qqq"
        harborServer='registry.cn-beijing.aliyuncs.com'
        projectName='test'
        imageUrl="${harborServer}/bjkubernetes/${projectName}"
        imageTag="${BUILD_ID}"
        //harborUserName="xiaoming"
        //harborPassword="Magedu2024"
    }
    stages {
        stage('Source') {
            steps {
                git branch: 'main', credentialsId: "${credential}", url: "${codeRepo}"
            }
        }
        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests clean package'
            }
        }
        stage('Test') {
           steps {
                  //注意:不要修改hello()函数,否则会导致下面失败
                sh 'mvn test'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build . -t "${imageUrl}:${imageTag}"'
            }           
        }
        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'harbor-passwd', \
                        passwordVariable: 'harborPassword', usernameVariable: 'harborUserName')]) {
                    //sh "echo ${harborPassword} | docker login -u ${env.harborUserName} --password-stdin ${harborServer}"
                    sh "docker login -u ${env.harborUserName} -p ${harborPassword} ${harborServer}"
                    sh "docker push ${imageUrl}:${imageTag}"
                    echo "username=${env.harborUserName}"
                    echo "password=${harborPassword}"
                }
            }   
        }
        stage('Run Docker ') {
            steps {
                //sh 'ssh root@10.0.0.101 "docker rm -f ${projectName} ; docker run --name ${projectName} -p 80:8888 -d ${imageUrl}:${imageTag}"'
                //sh 'ssh root@10.0.0.102 "docker rm -f ${projectName} ; docker run --name ${projectName} -p 80:8888 -d ${imageUrl}:${imageTag}"'
                sh "docker -H 10.0.0.101 rm -f ${projectName} ; docker -H 10.0.0.101 run --name ${projectName} -p 80:8888 -d ${imageUrl}:${imageTag}"
                sh "docker -H 10.0.0.102 rm -f ${projectName} ; docker -H 10.0.0.102 run --name ${projectName} -p 80:8888 -d ${imageUrl}:${imageTag}"
          
            }   
        }  		
    }
}
```



```
pipeline {
    agent any
    tools {
        maven 'Apache Maven 3.6.3'
    }
    environment {
        codeRepo="git@git.git.git:china/hello.git"
        credential="qqq"
        harborServer='registry.cn-beijing.aliyuncs.com'
        projectName='test'
        imageUrl="${harborServer}/bjkubernetes/${projectName}"
        imageTag="${BUILD_ID}"
        //harborUserName="xiaoming"
        //harborPassword="Magedu2024"
    }
    stages {
        stage('Source') {
            steps {
                git branch: 'main', credentialsId: "${credential}", url: "${codeRepo}"
            }
        }
        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests clean package'
            }
        }
        stage('Test') {
           steps {
                  //注意:不要修改hello()函数,否则会导致下面失败
                sh 'mvn test'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build . -t "${imageUrl}:${imageTag}"'
            }           
        }
        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'harbor-passwd', \
                        passwordVariable: 'harborPassword', usernameVariable: 'harborUserName')]) {
                    //sh "echo ${harborPassword} | docker login -u ${env.harborUserName} --password-stdin ${harborServer}"
                    sh "docker login -u ${env.harborUserName} -p ${harborPassword} ${harborServer}"
                    sh "docker push ${imageUrl}:${imageTag}"
                    echo "username=${env.harborUserName}"
                    echo "password=${harborPassword}"
                }
            }   
        }
        stage('Run Docker ') {
            steps {
                 sh "docker -H 10.0.0.203 rm -f ${projectName} ; docker -H 10.0.0.203 run --name ${projectName} -p 80:8888 -d ${imageUrl}:${imageTag}"
          
            }   
        }  		
    }
}
```



### Jenkinsfile

把jenkinsfile放在gitlab源码中

![image-20240315201734170](../笔记图片/image-20240315201734170.png)



#### 案例：参数选项和密码

```
pipeline {
    agent any
    parameters {
        string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')

        text(name: 'BIOGRAPHY', defaultValue: '', description: 'Enter some information about the person')

        booleanParam(name: 'TOGGLE', defaultValue: true, description: 'Toggle this value')

        choice(name: 'CHOICE', choices: ['One', 'Two', 'Three'], description: 'Pick something')

        password(name: 'PASSWORD', defaultValue: 'SECRET', description: 'Enter a password')
    }
    stages {
        stage('Example') {
            steps {
                echo "Hello ${params.PERSON}"

                echo "Biography: ${params.BIOGRAPHY}"

                echo "Toggle: ${params.TOGGLE}"

                echo "Choice: ${params.CHOICE}"

                echo "Password: ${params.PASSWORD}"
            }
        }
    }
}
```







### 企业级超级综合案例之打通下水道之国防代码检查之防空警告

![image-20240316100036408](../笔记图片/image-20240316100036408.png)

```bash
#wcaht webhook
https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=f81f24f1-c603-4a49-800c-76497f42400e
```

![image-20240316095736093](../笔记图片/image-20240316095736093.png)

#### jenkins安装插件

```
GitLab ,pipeline,Pipeline Stage View,blue Ocean,Qy Wechat Notification
```

#### 在 Jenkins 创建凭据连接 Harbor

![image-20240316100332485](../笔记图片/image-20240316100332485.png)

#### 在 Jenkins 安装Maven 工具

##### 在 Jenkins 上配置 Maven 环境

Mange Jenkins --- Global Tool configuration -- Maven 
根据上面的 mvn -version 命令结果,填写 MAVEN_HOME 的路径

```bash

root@ubuntu2204 ~]#apt update && apt -y install maven 
[root@ubuntu2204 ~]#vim /etc/maven/settings.xml
......
   <mirror>
       <id>nexus-aliyun</id>
       <mirrorOf>*</mirrorOf>
       <name>Nexus aliyun</name>
       <url>http://maven.aliyun.com/nexus/content/groups/public</url>
   </mirror>
 </mirrors>
.....
#查看Maven信息,记录执行结果供以下步骤使用
[root@ubuntu2204 ~]#mvn -version
Apache Maven 3.6.3
Maven home: /usr/share/maven
Java version: 11.0.17, vendor: Ubuntu, runtime: /usr/lib/jvm/java-11-openjdkamd64
Default locale: zh_CN, platform encoding: UTF-8
OS name: "linux", version: "5.15.0-52-generic", arch: "amd64", family: "unix
```

![image-20240316100419883](../笔记图片/image-20240316100419883.png)



#### 在 Jenkins 创建连接 GitLab 的凭据

![image-20240316100722411](../笔记图片/image-20240316100722411.png)

#### 在 Jenkins 上修改 Docker 的 socket 文件权限

默认Jenkins以Jenkins用户启动，无法访问Docker,构建时会提示如下错误:

解决错误

```bash
root@ubuntu2204 src]#ll /var/run/docker.sock
srw-rw---- 1 root docker 0  1月 31 17:20 /var/run/docker.sock=
[root@ubuntu2204 src]#cd 
[root@ubuntu2204 ~]#id jenkins
用户id=113(jenkins) 组id=118(jenkins) 组=118(jenkins)
[root@ubuntu2204 ~]#usermod -G docker jenkins
[root@ubuntu2204 ~]#id jenkins
用户id=113(jenkins) 组id=118(jenkins) 组=118(jenkins),119(docker)
[root@ubuntu2204 ~]#systemctl restart jenkins
```



#### 安装 SonarQube 并创建用户和令牌

安装 Sonarqueb Server ,并在 Sonarqube 创建用户并授权

```
#jenkins要用这个token连接sonarqub的api调用它执行代码检测操作
squ_a4afec5135f16f1f14e1517969f906dee702dd4b
```

![image-20240316101159426](../笔记图片/image-20240316101159426.png)

![image-20240316101237083](../笔记图片/image-20240316101237083.png)

![image-20240316101302718](../笔记图片/image-20240316101302718.png)





#### 在 SonarQube 添加 Jenkins 的回调接口

在SonarQube上添加webhook(网络调用),以便于Jenkins通过SonarQube Quality Gate插件调用其"质量 阈"信息,决定是否继续执行下面的构建步骤

 配置 --- 网络调用 webhook --- 创建

![image-20240316101427732](../笔记图片/image-20240316101427732.png)

输入名称和下面 Jenkins的URL地址

密码防止攻击使用, 可以随意输入

此处的密码没有复杂度和长度要求，但建议使用符合安全的密码

```bash
#确保soarnaqube能解析到jenkins的地址
http://www.jenkins.jenkins:8080/sonarqube-webhook
#生成Secet
openssl rand -base64 21
PiTVO6epEqIUlvU1DBoK1bVS3bkh
```

![image-20240316101912508](../笔记图片/image-20240316101912508.png)



#### 在 Jenkins 安装 SonarQube Scanner 插件

![image-20240316102337989](../笔记图片/image-20240316102337989.png)

#### 在 Jenkins 上配置系统的 SonarQube 服务器信息

Manage Jenkins -- Configure System

注意:Name名称区分大小写 

注意：http://www.SonarQube.sqb:9000 地址最后不能加/ 

注意域名解析

![image-20240316103133313](../笔记图片/image-20240316103133313.png)



#### 在 Jenkins 创建访问 Sonarqube的令牌凭据

jenkins要连接sonarqube提交代码质量检测

用前面小节生成的Sonarqube中用户令牌,在Jenkins 创建凭据

```
#添加token
squ_a4afec5135f16f1f14e1517969f906dee702dd4b
#指定凭据的ID
jenkins-to-sonarqube
```

![image-20240316102948827](../笔记图片/image-20240316102948827.png)

#### 在 Jenkins 上安装 SonarQube Scanner 命令工具

![image-20240316103429839](../笔记图片/image-20240316103429839.png)

#### 微信机器人

```bash
#wcaht webhook
https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=f81f24f1-c603-4a49-800c-76497f42400e
```

![image-20240316103819732](../笔记图片/image-20240316103819732.png)





#### 创建项目

![image-20240316103957468](../笔记图片/image-20240316103957468.png)



```
pipeline {
    agent any
    tools {
        maven 'Apache Maven 3.6.3'
    }
    environment {
        codeRepo="git@git.git.git:china/hello.git"
        credential='qqq'
        harborServer='registry.cn-beijing.aliyuncs.com'
        projectName='test'
        imageUrl="${harborServer}/bjkubernetes/${projectName}"
        imageTag="${BUILD_ID}"
    }
    stages {
        stage('Source') {
            steps {
                git branch: 'main', credentialsId: "${credential}", url: "${codeRepo}"
            }
        }
        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('Sonarqube') {#这里要写我配置的名字，这里存的就是sqb的地址和认证token
                    sh 'mvn sonar:sonar -Dsonar.java.binaries=target/'
                }
            }
        }
        stage("Quality Gate") {
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }		
        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests clean package'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker image build . -t "${imageUrl}:${imageTag}"'
                // input(message: '镜像已经构建完成，是否要推送？')
            }           
        }
        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'harbor-passwd', passwordVariable: 'harborPassword', usernameVariable: 'harborUserName')]) {
                    sh "docker login -u ${env.harborUserName} -p ${env.harborPassword} ${harborServer}"
                    sh "docker image push ${imageUrl}:${imageTag}"
                }
            }   
        }
        stage('Run Docker') {
            steps {
                //sh 'ssh root@10.0.0.202 "docker rm -f ${projectName} && docker run --name ${projectName} -p 80:8888 -d ${imageUrl}:${imageTag}"'
                //sh 'ssh root@10.0.0.203 "docker rm -f ${projectName} && docker run --name ${projectName} -p 80:8888 -d ${imageUrl}:${imageTag}"'
                sh "docker -H 10.0.0.203:2375 rm -f ${projectName} && docker -H 10.0.0.203:2375 run --name ${projectName} -p 80:8888 -d ${imageUrl}:${imageTag}"
                //sh "docker -H 10.0.0.102:2375 rm -f ${projectName} && docker -H 10.0.0.102:2375 run --name ${projectName} -p 80:8888 -d ${imageUrl}:${imageTag}"
            }   
        }         
        
    }
  post {
        success {
            mail to: 'root@wangxiaochun.com',
            subject: "Status of pipeline: ${currentBuild.fullDisplayName}",
            body: "${env.BUILD_URL} has result ${currentBuild.result}"
        }
        failure{
            qyWechatNotification failNotify: true, webhookUrl: https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=f81f24f1-c603-4a49-800c-76497f42400e'
        }
   } 
}
```

