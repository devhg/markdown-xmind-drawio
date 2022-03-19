Redis Cluster采用无中心结构，每个节点保存数据和整个集群状态，每个节点都和其他所有节点连接。

Redis Cluster提供了一种运行Redis安装的方法，在该安装中，数据会在多个Redis节点之间自动分片。Redis Cluster 在分区期间还提供了一定程度的可用性，这实际上是在某些节点出现故障或无法通信时继续操作的能力。但是，如果发生较大故障（例如，大多数主服务器不可用时），集群将停止运行。Redis Cluster自动分割在多个节点之间的数据集，一部分节点出现故障或无法与集群的其余部分通信时，继续运行的能力。



## 自建bridge网络，手动分配IP安装

### 1. 手动创建安装

需要手动创建网卡，指定ip启动多容器，然后进入一个容器执行命令创建集群。

具体安装步骤参考： https://www.stephen520.cn/blog/10271



### 2. docker-compose和脚本一键启动

基于上面手动安装过程，写了一些生成配置脚本。可以用docker-compose自动创建指定网段的网卡，自动创建多集群等功能。

配置文件模板 redis-cluster.tmpl。

```
port 6379
bind 0.0.0.0
protected-mode no
daemonize no
appendonly yes
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 15000
cluster-announce-ip 172.19.0.1${NO}
cluster-announce-port 6379
cluster-announce-bus-port 16379
```

注意：这里cluster-announce-ip为自动创建网卡的网段。${NO}主要是为了根据下面脚本，生成不同的IP。



生成配置文件脚本 generate.sh

生成6个实例配置文件，3主3从。IP分别是172.19.0.11、172.19.0.12、... 、172.19.0.16

```sh
for no in `seq 1 6`; do \
  mkdir -p redis-${no}/conf \
  && NO=${no} envsubst < redis-cluster.tmpl > redis-${no}/conf/redis.conf \
  && mkdir -p redis-${no}/data;\
done
```



docker-compose.yaml

注意：172.19.0.0/16网段已经存在，会出现创建失败。

参考 docker-compose手工指定IP https://www.cnblogs.com/xuanmanstein/p/10576476.html文章。可以执行 docker network prune删掉无用的网络，释放网段。

```yaml
# 描述 Compose 文件的版本信息
version: "3.8"

# 定义服务，可以多个
services:
  redis-cluster:
    image: redis:latest
    networks:
      redis:
        ipv4_address: 172.19.0.2
    command: redis-cli --cluster create 172.19.0.11:6379 172.19.0.12:6379 172.19.0.13:6379 172.19.0.14:6379 172.19.0.15:6379 172.19.0.16:6379 --cluster-replicas 1  --cluster-yes
    depends_on:
      - redis-1
      - redis-2
      - redis-3
      - redis-4
      - redis-5
      - redis-6

  redis-1: # 服务名称
    image: redis:latest # 创建容器时所需的镜像
    container_name: redis-1 # 容器名称
    restart: "no" # 容器总是重新启动
    networks:
      redis:
        ipv4_address: 172.19.0.11
    ports:
      - "6379:6379"
      - "16379:16379"
    volumes: # 数据卷，目录挂载
      - ./etc_rc.local:/etc/rc.local
      - ./redis-1/conf/redis.conf:/etc/redis/redis.conf
      - ./redis-1/data:/data
    command: redis-server /etc/redis/redis.conf # 覆盖容器启动后默认执行的命令
  redis-2:
    image: redis:latest
    container_name: redis-2
    networks:
      redis:
        ipv4_address: 172.19.0.12
    ports:
      - "6380:6379"
      - "16380:16379"
    volumes:
      - ./etc_rc.local:/etc/rc.local
      - ./redis-2/conf/redis.conf:/etc/redis/redis.conf
      - ./redis-2/data:/data
    command: redis-server /etc/redis/redis.conf
  redis-3:
    image: redis:latest
    container_name: redis-3
    networks:
      redis:
        ipv4_address: 172.19.0.13
    ports:
      - "6381:6379"
      - "16381:16379"
    volumes:
      - ./etc_rc.local:/etc/rc.local
      - ./redis-3/conf/redis.conf:/etc/redis/redis.conf
      - ./redis-3/data:/data
    command: redis-server /etc/redis/redis.conf
  redis-4:
    image: redis:latest
    container_name: redis-4
    networks:
      redis:
        ipv4_address: 172.19.0.14
    ports:
      - "6382:6379"
      - "16382:16379"
    volumes:
      - ./etc_rc.local:/etc/rc.local
      - ./redis-4/conf/redis.conf:/etc/redis/redis.conf
      - ./redis-4/data:/data
    command: redis-server /etc/redis/redis.conf
  redis-5:
    image: redis:latest
    container_name: redis-5
    networks:
      redis:
        ipv4_address: 172.19.0.15
    ports:
      - "6383:6379"
      - "16383:16379"
    volumes:
      - ./etc_rc.local:/etc/rc.local
      - ./redis-5/conf/redis.conf:/etc/redis/redis.conf
      - ./redis-5/data:/data
    command: redis-server /etc/redis/redis.conf
  redis-6:
    image: redis:latest
    container_name: redis-6
    networks:
      redis:
        ipv4_address: 172.19.0.16
    ports:
      - "6384:6379"
      - "16384:16379"
    volumes:
      - ./etc_rc.local:/etc/rc.local
      - ./redis-6/conf/redis.conf:/etc/redis/redis.conf
      - ./redis-6/data:/data
    command: redis-server /etc/redis/redis.conf

# 自动创建网络，并手动指定IP网段
networks:
  redis:
    ipam:
      config:
        - subnet: 172.19.0.0/16
```





## HOST模式，单独IP，不同端口安装

参考文章：

* https://www.hwholiday.com/2021/docker_redis/

* https://juejin.cn/post/6987764972411420709



### 1. docker-compose和一键脚本启动

和上面类似，但也有些区别，写了一些生成配置脚本。

**主要区别就是这里使用docker自带的bridge网络，使用不同端口来创建集群。**

配置文件模板 redis-cluster.tmpl。

```
port 6379
bind 0.0.0.0
protected-mode no
daemonize no
appendonly yes
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 15000
cluster-announce-ip 172.17.0.1
cluster-announce-port ${PORT}
cluster-announce-bus-port 1${PORT}

```

注意：这里cluster-announce-ip为bridge网卡的IP。使用不同的`cluster-announce-port ${PORT}  cluster-announce-bus-port 1${PORT}`



生成配置文件脚本 generate.sh

生成6个实例配置文件，3主3从。172.17.0.1:6379、172.17.0.1:6380、...、172.17.0.1:6384下不同端口通信

```sh
for port in `seq 6379 6384`; do \
  mkdir -p ${port}/conf \
  && PORT=${port} envsubst < redis-cluster.tmpl > ${port}/conf/redis.conf \
  && mkdir -p ${port}/data;\
done
```



docker-compose.yaml

```yaml
# 描述 Compose 文件的版本信息
version: "3.8"

# 定义服务，可以多个
services:
  redis-cluster:
    image: redis:latest
    command: redis-cli --cluster create 172.17.0.1:6379 172.17.0.1:6380 172.17.0.1:6381 172.17.0.1:6382 172.17.0.1:6383 172.17.0.1:6384 --cluster-replicas 1  --cluster-yes
    depends_on:
      - redis-6379
      - redis-6380
      - redis-6381
      - redis-6382
      - redis-6383
      - redis-6384
  redis-6379: # 服务名称
    image: redis:latest # 创建容器时所需的镜像
    container_name: redis-6379 # 容器名称
    restart: "no" # 容器总是重新启动
    ports:
      - "6379:6379"
      - "16379:16379"
    volumes: # 数据卷，目录挂载
      - ./etc_rc.local:/etc/rc.local
      - ./6379/conf/redis.conf:/etc/redis/redis.conf
      - ./6379/data:/data
    command: redis-server /etc/redis/redis.conf # 覆盖容器启动后默认执行的命令

  redis-6380:
    image: redis:latest
    container_name: redis-6380
    ports:
      - "6380:6379"
      - "16380:16379"
    volumes:
      - ./etc_rc.local:/etc/rc.local
      - ./6380/conf/redis.conf:/etc/redis/redis.conf
      - ./6380/data:/data
    command: redis-server /etc/redis/redis.conf

  redis-6381:
    image: redis:latest
    container_name: redis-6381
    ports:
      - "6381:6379"
      - "16381:16379"
    volumes:
      - ./etc_rc.local:/etc/rc.local
      - ./6381/conf/redis.conf:/etc/redis/redis.conf
      - ./6381/data:/data
    command: redis-server /etc/redis/redis.conf

  redis-6382:
    image: redis:latest
    container_name: redis-6382
    ports:
      - "6382:6379"
      - "16382:16379"
    volumes:
      - ./etc_rc.local:/etc/rc.local
      - ./6382/conf/redis.conf:/etc/redis/redis.conf
      - ./6382/data:/data
    command: redis-server /etc/redis/redis.conf

  redis-6383:
    image: redis:latest
    container_name: redis-6383
    ports:
      - "6383:6379"
      - "16383:16379"
    volumes:
      - ./etc_rc.local:/etc/rc.local
      - ./6383/conf/redis.conf:/etc/redis/redis.conf
      - ./6383/data:/data
    command: redis-server /etc/redis/redis.conf

  redis-6384:
    image: redis:latest
    container_name: redis-6384
    ports:
      - "6384:6379"
      - "16384:16379"
    volumes:
      - ./etc_rc.local:/etc/rc.local
      - ./6384/conf/redis.conf:/etc/redis/redis.conf
      - ./6384/data:/data
    command: redis-server /etc/redis/redis.conf

# 使用已经存在的bridge网络，或者也可以手动创建一个 替换上面的172.17.0.1
# 创建方式：docker network create redis --subnet 172.28.0.0/16
# 当然也有另一种方式，就是用docker-compose自动创建的网络，不过需要手动进入一个容器运行 cluster 创建命令
networks:
  persist:
    external:
      name: bridge
```









## 其他参考资料

docker容器如何与宿主机通信 https://www.cnblogs.com/koushr/p/14559481.html