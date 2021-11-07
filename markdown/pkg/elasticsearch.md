docker安装elasticsearch及其客户端

本文将主要记录elasticsearch安装过程中遇到的一些问题

## 1. 安装配置elasticsearch

### 1.1 docker下载相关镜像

```bash
docker pull elasticsearch:7.6.2
```

### 1.2 创建持久化文件

```bash
mkdir -p data/elasticsearch/config
mkdir -p data/elasticsearch/data
```

初始化配置

```bash
echo "http.host: 0.0.0.0" >> data/elasticsearch/config/elasticsearch.yml
```

注意这里“：”后的空格，否则会报语法错误

### 1.3 启动镜像

```bash
docker run --name elasticsearch -p 9200:9200  -p 9300:9300 \
 -e "discovery.type=single-node" \
 -e ES_JAVA_OPTS="-Xms84m -Xmx512m" \
 -v data/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
 -v data/elasticsearch/data:/usr/share/elasticsearch/data \
 -v data/elasticsearch/plugins:/usr/share/elasticsearch/plugins \
 -d elasticsearch:7.6.2
```

-p:端口映射

-e discovery.type=single-node 单点模式启动

-e ES_JAVA_OPTS="-Xms84m -Xmx512m"：设置启动占用的内存范围（实验环境启动后可能因为云服务器内存过小而占满）
 -v 目录挂载
 -d 后台运行

启动后使用docker ps查看发现未正常启动，查看启动日志

```bash
docker logs elasticsearch
```





## 2. 安装使用可视化工具

### 2.1 下载同版本镜像

```bash
docker pull kibana:7.6.2
```

### 2.2 初始化配置

```bash
mkdir -p data/kibana
touch data/kibana/kibana.yml
vim data/kibana/kibana.yml
```

```yml
server.host: 0.0.0.0
elasticsearch.hosts: http://你的ip:9200
```

保存退出



### 2.3 启动kibana

```bash
docker run --name kibana -v /mydata/kibana/kibana.yml:/usr/share/kibana/config/kibana.yml -p 5601:5601 -d kibana:7.6.2 
```

启动后可通过docker ps查看是否启动成功，等待一段时间后浏览器访问 ip:5601 进行测试