# docker redis-cluster 
使用docker部署redis-cluster集群

---
#### 安装docker与docker-compose
   
+ [安装docker](https://docs.docker.com/engine/)
+ [安装docker-compose](https://docs.docker.com/compose/) 

docker-compose是可选的，但它会更方便地管理多个docker containter。   

----

#### redis镜像

本实例中使用的redis版本为5.0.8。

`docker pull redis:5.0.8`

其余的版本参考[dockerhub](https://hub.docker.com/_/redis)

----

#### 实验环境 

+ Linux server1 (ubuntu18.04) - 192.168.56.101
+ Linux server2 (ubuntu18.04) - 192.168.56.102
+ 一台可以访问局域网的电脑 - 191.168.56.1

---

#### 模拟

在192.168.56.101上运行多个redis实例，用于模拟多台机器的redis实例。

redisz主从集群**最少需要6个节点**,在端口：7000  7001  7002  7100  7101  7102上分别运行一个redis实例。

步骤：

+ 创建一个单独的文件夹用于测试
```
mkdir redis-cluster
cd redis-cluster
```
+ 为每一个redis实例创建配置文件夹

`mkdir 7000 7001 7002 7100 7101 7102`
+ 以7000端口为示例配置

```cd 7000```
+ conf存放redis运行配置文件，data存放redis数据库以及其它数据文件

```mkdir conf data```
+ 创建一份redis配置redis.conf，可从官网下载，或者直接使用本仓库的redis.conf文件

```
cd conf
touch redis.conf
```
+ 运行起一个redis服务

参考官方文档：[Redis cluster tutorial](https://redis.io/topics/cluster-tutorial/)

在redis.conf中几个参数需要配置：

```
# cluster-enabled<yes/no>: 必须调整为yes，否则就是一个单独的redis服务，无法作为集群的节点
cluster-enabled yes
# cluster-config-file <filename>: 集群运行时，生成一个存放节点信息的配置文件“nodes.conf”
# redis自动生成 无需人为编辑，可以查看。
# 要删除集群，可通过删除每个节点下的./data/nodes.conf实现。
cluster-config-file nodes.conf
# cluster-node-timeout <milliseconds>: 节点之间通信最长无响应时间（毫秒）
cluster-node-timeout 15000
# appendonly
appendonly yes 
# 密码 - 每一个节点的 masterauth 必须相同
requirepass 123456
masterauth 123456789
# 端口
port 7000
```
+ 将redis.conf的拷贝到其余的文件夹下，目录结构保持一致，除了端口port外内容一致。

+ 编写yml文件 - docker-compose-redis-cluster.yml

```yaml
version: "3.4"                                           
                                                         
x-image:                                                 
  &default-image                                         
  redis:5.0.8                                           
x-restart:                                               
  &default-restart                                       
  always                                                 
x-network_mode:                                          
  &default-network_mode                                  
  host                                                   
                                                         
services:                                                
  redis_7000:                                            
    image: *default-image                                
    container_name: redis_7000                           
    command: redis-server /etc/redis/redis.conf          
    volumes:                                             
      - "./7000/data:/data"                              
      - "./7000/conf/redis.conf:/etc/redis/redis.conf"   
    network_mode: *default-network_mode                  
    restart: *default-restart                            

  redis_7002:
    image: *default-image
    container_name: redis_7002
    command: redis-server /etc/redis/redis.conf
    volumes:
      - "./7002/data:/data"
      - "./7002/conf/redis.conf:/etc/redis/redis.conf"
    network_mode: *default-network_mode
    restart: *default-restart

  redis_7100:
    image: *default-image
    container_name: redis_7100
    command: redis-server /etc/redis/redis.conf
    volumes:
      - "./7100/data:/data"
      - "./7100/conf/redis.conf:/etc/redis/redis.conf"
    network_mode: *default-network_mode
    restart: *default-restart

  redis_7101:
    image: *default-image
    container_name: redis_7101
    command: redis-server /etc/redis/redis.conf
    volumes:
      - "./7101/data:/data"
      - "./7101/conf/redis.conf:/etc/redis/redis.conf"
    network_mode: *default-network_mode
    restart: *default-restart

  redis_7102:
    image: *default-image
    container_name: redis_7102
    command: redis-server /etc/redis/redis.conf
    volumes:
      - "./7102/data:/data"
      - "./7102/conf/redis.conf:/etc/redis/redis.conf"
    network_mode: *default-network_mode
    restart: *default-restart
```

注意：
    
    docker redis-cluster节点间的通信，需要网络模式为"host"。

+ 运行redis实例

`docker-compose -f docker-compose-redis-cluster.yml up -d`

此时在端口 7000  7001  7002  7100  7101  7102上分别运行一个redis实例。

可以查看一下各个redis containter的log。

至此6个独立的redis服务已运行起来，还没有"集群"。

----

+ 创建集群

redis5使用redis-cli来设置集群。

`docker run --rm -it --net=host redis:5.0.8 redis-cli --cluster create 192.168.56.101:7000 192.168.56.101:7001 192.168.56.101:7002 192.168.56.101:7100 192.168.56.101:7101 192.168.56.101:7102 --cluster-replicas 1`
 
 会自动生成配置信息。这里服务运行在”192.168.56.101“上，看自身机器的局域网ip相应修改。
 另外：因为有人说因为服务需要密码，所以在创建集群的时候需要加上' -a <requirepass> '参数。这里应该是'-a 123456'。
 但是测试发现加上之后依然报错：(error) NOAUTH Authentication required. 跟不加是一样的效果。
 但都不妨碍请求访问。所以就都不加了。
 
集群创建完成。访问任何节点，都可以取到集群中全部的redis值。

+ 新增节点

192.168.56.101机器上7000  7001  7002  7100  7101  7102模拟集群ok，如何添加局域网中的另一台服务器的redis节点呢？

两步：新增一台空的redis服务节点到集群，然后分配一些数据。

首先在192.168.56.102服务器上运行两个redis在端口： 7000（master） 7100（slaver），同上。
然后在局域网内的任意一台客户端上执行： 

`redis-cli --cluster add-node  -a 123456 192.168.56.102:7000 192.168.56.101:7000`
`redis-cli --cluster add-node  -a 123456 192.168.56.102:7100 192.168.56.101:7000`

如果没有报错说明节点新增成功。

此时需要为新增的节点分配hash槽，以便新节点可以存储数据.
redis-cluster一共有16384个槽位，如果分4个配套master节点：每个节点是4096个槽位.

`redis-cli --cluster reshard 192.168.56.101:7000`

会列出当前所有节点id与节点主机与端口信息。这时候问需要分配多少hash槽位？

根据 16384 / master节点个数计算或者自定义都可。

问需要分配给哪个槽位id？

将192.168.56.102:7000前的id复制粘贴上即可。

问需要从那些master节点迁移槽位过来？

all  # 全部

或者

依次输入现存master节点的id，最后输入 done

+ 确认迁移无误

```
root@home:/data# redis-cli -c -p 7000 -h 192.168.56.102
192.168.56.102:7000> auth 123456
OK
192.168.56.102:7000> cluster nodes
be27844b773d12c473991ca952b8fdb5991a2d57 192.168.56.101:7001@17001 master - 0 1586702967000 2 connected 6827-10922
04a19473ae91a2637fe6ba5ea8e878a310dce3fb 192.168.56.101:7102@17102 slave be27844b773d12c473991ca952b8fdb5991a2d57 0 1586702969892 2 connected
7075bb9c83a793727c8a088dca7278c2b7e0e3e5 192.168.56.101:7101@17101 slave 27edc2c0d70d848e69f111f27c95249ee70de7f0 0 1586702968000 1 connected
27edc2c0d70d848e69f111f27c95249ee70de7f0 192.168.56.101:7000@17000 master - 0 1586702970895 1 connected 1365-5460
12d9e1c7b537aba3b5273b57e0d74e76c4ab6da2 192.168.56.101:7100@17100 slave dbb38426df22e20ec482862ff06af8cd7aa08413 0 1586702968000 3 connected
dbb38426df22e20ec482862ff06af8cd7aa08413 192.168.56.101:7002@17002 master - 0 1586702969000 3 connected 12288-16383
710bc2cff9b1132a44249bd7b5be86afa9ad26bf 192.168.56.102:7000@17000 myself,master - 0 1586702969000 8 connected 0-1364 5461-6826 10923-12287
3566c95b76ecb88b38a8d5ed036f64af3d468162 192.168.56.102:7100@17100 master - 0 1586702968889 7 connected
192.168.56.102:7000>
```

+ 
