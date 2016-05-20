Docker install
依赖性检查--内核版本必须大于3.10
#uname -r 3.10.0-229.el7.x86_64 
使用yum安装
使用有sudo权限的帐号登录系统。
更新yum包。
$ sudo yum update 
添加docker源。
$ cat >/etc/yum.repos.d/docker.repo <<-EOF
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7 enabled=1 gpgcheck=1 gpgkey=https://yum.dockerproject.org/gpg EOF
使用yum命令安装docker。
$ sudo yum install docker-engine 
启动docker服务。
$ sudo service docker start
确认docker是否安装成功。
$ sudo docker run hello-world

二.本地Registry部署过程
1.使用docker的registry镜像部署[非本次部署方法]
从Docker Hub上拉取已经构建好的镜像registry
#docker pull registry
创建本地registry
#docker run -p 5000:5000 -d -i -t registry

2.使用registry v2(可以通过docker实现，也可以编译成一个二进制程序，直接运行)
#nohup ./registry server config.yml &
如下为config.yml配置文件内容
version: 0.1
log:
fields:
service: registry
storage:
cache:
blobdescriptor: inmemory
filesystem:
rootdirectory: /docker/registry
http:
addr: :5000
headers:
X-Content-Type-Options: [nosniff]
health:
storagedriver:
enabled: true
interval: 10s
threshold: 3

三.客户端连接Registry相关修改
仓库搭建完成后，因为Docker从1.3.X之后，与docker registry交互默认使用的是https，然而此处搭建的私有仓库只提供http服务，所以当与私有仓库交互时就会报错误:
Error: Invalid registry endpoint
为了解决这个问题需要在启动docker server时增加启动参数为默认使用http访问。修改docker配置文件

1.针对1.9.1版本 直接修改 /etc/sysconfig/docker
#vi /etc/sysconfig/docker
编辑新增如下内容：
OPTIONS='--selinux-enabled --insecure-registry 10.88.60.100:5000'

2.针对1.10.2版本 因为没有 /etc/sysconfig/docker这个文件，所以需要先编辑 /lib/systemd/system/docker.service 这个文件，将配置文件路径写进去
#vi /lib/systemd/system/docker.service
新增如下内容：
EnvironmentFile=/etc/sysconfig/docker
然后新增一个 /etc/sysconfig/docker，将如下内容新增即可
OPTIONS='--selinux-enabled --insecure-registry 10.88.60.100:5000'

3.重启docker
systemctl daemon-reload
systemctl restart docker



四.本地搭建zabbix服务器
1.从本地registry[地址为：10.88.60.100:5000]下载zabbix Server与zabbix DB的镜像
#docker pull 10.88.60.100:5000/zabbix-3.0
#docker pull 10.88.60.100:5000/zabbix/zabbix-db-mariadb

#docker images
REPOSITORY TAG IMAGE ID CREATED VIRTUAL SIZE
10.88.60.100:5000/zabbix-3.0 latest 5183c6df9b26 13 days ago 754.3 MB
zabbix/zabbix-3.0 latest 5183c6df9b26 13 days ago 754.3 MB
10.88.60.100:5000/zabbix/zabbix-db-mariadb latest 8e71ac7247d1 2 weeks ago 554.4 MB
zabbix/zabbix-db-mariadb latest 8e71ac7247d1 2 weeks ago 554.4 MB

2.pull到本地的镜像名称为ip:5000/zabbix-3 可以通过docker rename 进行重命名

‘3’.启动一个数据卷容器--本例中没有进行该操作，直接挂载了本地数据卷给数据库容器
# create /var/lib/mysql as persistent volume storage
docker run -d -v /var/lib/mysql --name zabbix-db-storage busybox:latest
创建了一个名字叫做zabbix-db-storage的容器，挂载了 /var/lib/mysql 的数据卷，以备后面数据库启动时候引用

3.zabbix数据库容器启动脚本
*******************************************
docker run \
-d \
--name zabbix-db1 \
-v /backups:/backups \
-v /etc/localtime:/etc/localtime:ro \
-v /mysqldata/zabbix-db1:/var/lib/mysql \
--env="MARIADB_USER=zabbix" \
--env="MARIADB_PASS=my_password" \
zabbix/zabbix-db-mariadb
*******************************************
该命令 使用“zabbix/zabbix-db”这个镜像，启动了一个名字为‘zabbix-db1’的容器，在容器中分别挂载了本地目录/backups、/etc/localtime、/mysqldata/zabbix-db1至相应路径
同时设定了数据库的环境变量，用户名和密码

4.zabbix server容器启动脚本
*******************************************
docker run \
-d \
--name zabbix1 \
-p 80:80 \
-p 10051:10051 \
-v /etc/localtime:/etc/localtime:ro \
--link zabbix-db1:zabbix.db \
--env="ZS_DBHost=zabbix.db" \
--env="ZS_DBUser=zabbix" \
--env="ZS_DBPassword=my_password" \
zabbix/zabbix-3.0:latest
*******************************************
该命令，使用“zabbix/zabbix-3.0:latest”这个镜像，启动了一名字为‘zabbix1’的容器，在容器中映射了容器端口80和10051至宿主机的80和10051端口(可以不同)。同时在容器中挂载了
/etc/localtime这个目录，然后link了数据库服务器，保证应用可以访问到数据库，设定了环境变量

5.应用和数据库启动完成，可以通过网页http://宿主机ip+端口号 访问zabbix页面


五.容器之间连接-Links
Docker通过如下两种方式来暴露连接提供的服务。
1. 环境变量
    当两个容器通过link命令进行连接以后，Docker会在目标容器中设置相关的环境变量，以便在目标容器中使用源容器提供的服务，例如zabbix应用启动时，使用命令 --link zabbix-db1:zabbix.db，连接了zabbix-db1这个容器，连接别名为zabbix.db，容器启动后，命令行执行env，其中以别名开头的内容，均为设置的环境变量
#env
ZABBIX.DB_PORT_3306_TCP_ADDR=172.17.0.2
ZABBIX.DB_ENV_DB_innodb_log_file_size=128M
ZABBIX.DB_PORT_3306_TCP_PORT=3306
ZABBIX.DB_ENV_TERM=xterm
ZABBIX.DB_PORT_3306_TCP_PROTO=tcp
ZABBIX.DB_ENV_DB_innodb_flush_log_at_trx_commit=0
ZABBIX.DB_ENV_DB_query_cache_size=0
ZABBIX.DB_PORT=tcp://172.17.0.2:3306
ZABBIX.DB_ENV_DB_innodb_flush_method=O_DIRECT
ZABBIX.DB_ENV_MARIADB_PASS=my_password
2./etc/hosts
查看目标容器的/etc/hosts文件，可以看到，容器连接zabbix-db1的对应地址为172.17.0.2，该地址为zabbix-db1容器的地址，容器对zabbix.db连接的操作将会映射到该地址上
[root@b46d81b957bf /]# cat /etc/hosts
172.17.0.3 b46d81b957bf
127.0.0.1 localhost
::1 localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.17.0.2 zabbix.db 8d754c313d2c zabbix-db1


六.代理连接
通过代理连接，可以解耦两个原本直接相连的容器的耦合性。同时实现跨宿主机容器连接
1.启动一个redis-server的容器
#docker run -d --name redis crosbymichael/redis
2.建立一个代理容器ambassador2，连接到redis
#docker run -d --link redis:redis --name redis_ambassador -p 6379:6379 svendowideit/ambassador
3.在客户端主机上启动一个代理容器ambassador1，连接到ambassador2，其中-e的参数后面，需要增加的内容为ambassador2中的env环境变量内容？
#docker run -d --name redis_ambassador --expose 6379 -e REDIS_PORT_6379_TCP=tcp://192.168.1.52:6379 svendowideit/ambassador
4.客户端主机如果需要使用redis服务，只需要连接到本机的redis_ambassador这个容器即可
#docker run -i -t --rm --link redis_ambassador:redis relateiq/redis-cli redis 172.17.0.160:6379> ping PONG

七.为镜像容器创建ssh
使用docker attach/docker exec命令无法解决远程管理容器的需求，因此要为容器安装SSH服务
1.配置容器内系统的软件源，例如163源等
2.安装配置ssh服务
#yum install openssh-server


八.创建shipyard进行容器管理
http://shipyard-project.com/docs/deploy/manual/

自动部署(手动部署没有成功)
1.在shipyard-server上执行命令
#curl -sSL https://shipyard-project.com/deploy | bash -s
该命令会下载如下镜像，并启动如下容器，启动正常以后，就可以通过http://host-ip:8080访问shipyardWEBUI管理界面了，用户名admin，
密码shipyard。
IMAGES                                            CONTAINER NAME
shipyard/shipyard:latest         shipyard-controller
swarm:latest             shipyard-swarm-agent
swarm:latest              shipyard-swarm-manager
shipyard/docker-proxy:latest shipyard-proxy
alpine shipyard-certs
microbox/etcd:latest shipyard-discovery
rethinkdb shipyard-rethinkdb

2.在shipyard管理的node节点执行命令
#curl -sSL https://shipyard-project.com/deploy | ACTION=node DISCOVERY=etcd://10.88.142.201:4001 bash -s
该命令在node节点下载如下镜像，并启动如下容器，启动正常以后，就可以通过shipyard页面读取到docker engine的信息，包括镜像、容器、打开console、查看性能、查看容器日志等
IMAGES                                            CONTAINER NAME
swarm:latest                                     shipyard-swarm-agent
swarm:latest                                      shipyard-swarm-manager   (应该是没有用的，关闭了功能正常)
shipyard/docker-proxy:latest           shipyard-proxy
alpine                                                 shipyard-certs 



附录：
1. docker pull的registry镜像无法正常运行，可能是因为python进行了升级
2.容器启动以后，其挂载的volume和端口映射，修改都不是很容易
3.docker inspect命令在--format后面，{{.Config.Volumes}}  {{.Hostconfig.Links}}
4.数据卷会绕过拷贝写，提升本地磁盘IO性能的同时，不会被打包进docker commit的镜像中，dockerfile不支持挂载本地数据卷
5.数据卷容器指一个专门用于挂在数据卷的容器，主要用于多个容器需要从一处获取数据
6.数据卷容器：数据卷一旦声明，它的生命周期和声明它的容器就无关了，即使容器停止，数据卷也依然存在。如果要删除数据卷，需要删除所有依赖它的容器，并且在删除最后一个依赖容器的时候，加入-v参数
7.数据卷容器：利用其可以进行数据的备份和恢复
    声明：$ docker run -d -v /dbdata --name dbdata busybox:latest
    备份：$ docker run --volumes-from /dbdata --name container1 -v $(pwd):/backup ubuntu tar cvf /backup/backup.tar /dbdata
    恢复：$ docker run --volumes-from /dbdata --name container2 -v $(pwd):/backup busybox tar xvf /backup/backup.tar
8.容器连接依赖于容器的名字  源容器:目标容器:连接别名
9.docker tag containerID 10.88.60.100:5000/tag_name
   docker push 10.88.60.100:5000/tag_name
   docker pull 10.88.60.100:5000/tag_name
10.RUN vs CMD命令:Build时执行RUN，RUN时执行CMD，也就是说，CMD才是镜像最终执行的命令
11.CMD vs ENTRYPOINT:CMD命令是可覆盖的，docker run后面输入的命令与CMD指定的命令匹配时，会把CMD指定的命令替换成docker run中带的命令。而ENTRYPOINT指定的命令只是一个“入口”，docker run后面的内容会全部传给这个“入口”，而不是进行命令的替换
12.Docker实际上把所有东西都放到/var/lib/docker路径下了

