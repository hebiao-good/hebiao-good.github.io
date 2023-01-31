docker network create --subnet=172.72.12.0/24 fastcfs-network

# 有挂载目录
docker run -dit --name fast0 -v C:/111:/data --net=fastcfs-network --privileged=true  ubuntu /bin/bash
docker run -dit --name fast10 -v C:/pgsql_instance2:/opt --net=fastcfs-network --privileged=true  ubuntu /bin/bash
docker run -dit --name fast0 -v C:/pgsql_instance/data:/opt/fastcfs/fuse --net=fastcfs-network --privileged=true  ubuntu /bin/bash
# 无挂载目录
docker run -dit --name fast1  --net=fastcfs-network --privileged=true  ubuntu /bin/bash
docker run -dit --name fast2  --net=fastcfs-network --privileged=true  ubuntu /bin/bash

docker exec -it fast0 bash

1. 免密操作，先安装ssh
apt-get update
apt-get install sudo
sudo apt install vim
sudo apt-get install openssh-server

1.1 修改密码
su root
passwd

修改文件
vi /etc/ssh/sshd_config
将PermitRootLogin prohibit-password修改为
PermitRootLogin yes
重启
sudo service ssh restart

1.2 免密操作
ssh-keygen -t rsa
# 一路回车
ssh-copy-id 172.72.12.3


2. 配置 apt 存储库和签名密钥，以使系统的包管理器启用自动更新。
sudo apt-get install curl gpg
curl http://www.fastken.com/aptrepo/packages.fastos.pub | gpg --dearmor > fastos-archive-keyring.gpg
sudo install -D -o root -g root -m 644 fastos-archive-keyring.gpg /usr/share/keyrings/fastos-archive-keyring.gpg
sudo sh -c 'echo "deb [signed-by=/usr/share/keyrings/fastos-archive-keyring.gpg] http://www.fastken.com/aptrepo/fastos/ fastos main" > /etc/apt/sources.list.d/fastos.list'
rm -f fastos-archive-keyring.gpg

2.1 如果需要安装调试包，执行：
sudo sh -c 'echo "deb [signed-by=/usr/share/keyrings/fastos-archive-keyring.gpg] http://www.fastken.com/aptrepo/fastos-debug/ fastos-debug main" > /etc/apt/sources.list.d/fastos-debug.list'

2.2 然后更新包缓存
sudo apt update
3. fastDIR server安装
sudo apt install fastdir-server -y
查看目录 etc/fastcfs/fdir/

4. faststore server安装
sudo apt install faststore-server -y
查看目录 etc/fastcfs/fstore/

5. FastCFS客户端安装(所有机器都要)
在需要使用FastCFS存储服务的机器（即FastCFS客户端）上执行：
# 注：fastcfs-fused依赖fuse3
sudo apt install fastcfs-fused -y

6. Auth server安装（可选）
如果需要存储池或访问控制，则需要安装本模块。
在需要运行 Auth server的服务器上执行：
sudo apt install fastcfs-auth-server -y


7. 修改fdir集群配置文件
用root登录主控机172.72.12.2
vi /etc/fastcfs/fdir/cluster.conf
添加节点信息
[server-1]
host = 172.72.12.2 #节点1
[server-2]
host = 172.72.12.3 #节点2

7.1 修改fdir服务配置文件
vi etc/fastcfs/fdir/server.conf
server.conf基本上用默认配置即可，如果需要开启存储插件，
需要把[storage-engine]下面的enable=true，同时配置storage.conf。

7.2 复制cluster文件到其他节点
把cluster.conf配置文件通过scp命令复制到其他节点（包括客户端节点），server.conf不需要复制：
scp /etc/fastcfs/fdir/cluster.conf 172.72.12.3:/etc/fastcfs/fdir/cluster.conf
scp /etc/fastcfs/fdir/cluster.conf 172.72.12.4:/etc/fastcfs/fdir/cluster.conf


8. fstore 配置
注释：fstore是存储数据的核心组件，修改cluster配置文件，一定要了解fstore存储的基本原理。
了解SG（服务器分组），DG（数据分组），DGC（数据分组数）等几个名词的相互关系

用root登录主控机172.72.12.2
修改fstore集群配置文件
vi /etc/fastcfs/fstore/cluster.conf 
让fstore的集群拓扑为：1个SG，SG里面包含三台服务器，DG为2，DGC为128，修改后的内容如下：

# the group count of the servers / instances
server_group_count = 1 # SGC=1

# all data groups must be mapped to the server group(s) without omission.
# once the number of data groups is set, it can NOT be changed, otherwise
# the data access will be confused!
data_group_count = 64 #DGC

# config the auth config filename
auth_config_filename = ../auth/auth.conf

[group-cluster]
# the default cluster port
port = 21014

[group-replica]
# the default replica port
port = 21015

[group-service]
# the default service port
port = 21016

[server-group-1]
server_ids = [1, 2]
data_group_ids = [1, 64]

[server-1]
host = 172.72.12.2

[server-2]
host = 172.72.12.3


8.1 修改fdir服务配置文件
vi etc/fastcfs/fstore/server.conf
server.conf基本上用默认配置即可，如果需要修改存储目录，需要修改storage.conf，
如果有多个数据盘可以配置多个目录，充分利用硬盘空间

8.2 复制cluster文件到其他节点
fstore的cluster.conf,server.conf,storage.conf配置完成后，
把cluster.conf配置文件通过scp命令复制到其他节点，server.conf不需要复制：
scp /etc/fastcfs/fstore/cluster.conf 172.72.12.3:/etc/fastcfs/fstore/cluster.conf
scp /etc/fastcfs/fstore/cluster.conf 172.72.12.4:/etc/fastcfs/fstore/cluster.conf

8.3 FastCFS客户端配置
vi /etc/fastcfs/fcfs/fuse.conf
scp /etc/fastcfs/fcfs/fuse.conf 172.72.12.3:/etc/fastcfs/fcfs/fuse.conf


9. 集群启动
先下载systemctl
sudo apt install systemctl
然后用root用户分别登录到2个节点，执行如下命令：
sudo systemctl restart fastdir
sudo systemctl restart faststore

9.1 客户端启动
sudo apt install systemctl
如果三个节点的所有组件启动没有错误，在客户端节点可以启动客户端程序，
把fastCFS的默认pool fs挂载到相应目录。
以本次部署为例，root用户登录到客户端节点172.72.12.4，执行命令：
sudo systemctl restart fastcfs

查看日志文件/opt/fastcfs/fcfs/logs/fcfs_fused.log看下启动是否有误，
如果无误可以通过df -h查看挂载的目录 /opt/fastcfs/fuse

10. 验证集群状态（客户端执行）
10.1 查询fdir整个集群状态
fdir_cluster_stat
server_id: 1, host: 172.72.12.2:11012, status: 23 (ACTIVE), is_master: 1, data_version: 1
server_id: 2, host: 172.72.12.3:11012, status: 23 (ACTIVE), is_master: 0, data_version: 1

server count: 2

10.2 查询fdir某个节点状态，1表示是server编号，和cluster.conf里面的server-N相对应
fdir_service_stat 1
server_id: 1
        host: 172.72.12.2:11012
        status: 23 (ACTIVE)
        is_master: true
        connection : {current: 3, max: 4}
        data : {current_version: 1, confirmed_version: 1}
        binlog : {current_version: 1, writer: {next_version: 2, total_count: 1, waiting_count: 0, max_waitings: 0}}
        dentry : {current_inode_sn: 1000001, ns_count: 1, dir_count: 1, file_count: 0}

10.3 fstore集群状态，显示所有状态
fs_cluster_stat

11. dir，store，fastcfs的各自配置路径
cd /etc/fastcfs/fdir
cd /etc/fastcfs/fstore
cd /etc/fastcfs/fcfs

faststore数据存放路径
cd /opt/faststore/data
fastcfs的默认挂载路径
cd /opt/fastcfs/fuse

docker run -dit --name fast00 -v E:/fast0:/opt/fastcfs/fuse --net=fastcfs-network --privileged=true  ubuntu-fast0 /bin/bash
docker run -dit --name fast00 -v E:/fast0:/opt --net=fastcfs-network --privileged=true  ubuntu-fast0 /bin/bash
数据库实例挂载
docker run -dit --name fast00 -v C:/pgsql_instance/data:/opt/fastcfs/fuse --net=fastcfs-network --privileged=true  ubuntu-fast0 /bin/bash
docker run -dit --name fast10 -v C:/pgsql_instance2:/opt --net=fastcfs-network --privileged=true  ubuntu /bin/bash





1. 创建容器时，将宿主机的111目录和容器的根目录/opt进行挂载，后续运行fasfCFS客户端程序时会在/opt目录下生成/opt/fastcfs/fuse目录，但不同步

2. 先创建容器，运行fastcfs客户端程序以后，会在/opt目录下生成/opt/fastcfs/fuse目录，此时关闭容器，将容器生成镜像，
重新运行镜像时将宿主机的111目录挂载到/opt/fastcfs/fuse

3. 创建容器时，就将宿主机的111目录挂载到容器的/data目录上，然后进入容器运行fastCFS客户端程序，将/opt/fastcfs/fuse挂载到容器中的/data目录下














