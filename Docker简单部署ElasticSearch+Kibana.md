# Docker简单部署ElasticSearch+Kibana

[TOC]

## 一、ElasticSearch是什么

Elasticsearch也使用Java开发并使用Lucene作为其核心来实现所有索引和搜索的功能，但是它的目的是通过简单的RESTful API来隐藏Lucene的复杂性，从而让全文搜索变得简单。
不过，Elasticsearch不仅仅是Lucene和全文搜索，我们还能这样去描述它：

- 分布式的实时文件存储，每个字段都被索引并可被搜索
- 分布式的实时分析搜索引擎
- 可以扩展到上百台服务器，处理PB级结构化或非结构化数据

## 二、Docker部署ElasticSearch

### 2.1 拉取镜像

```
docker pull docker.elastic.co/elasticsearch/elasticsearch:6.2.2
```



### 2.2 运行容器

`ElasticSearch`的默认端口是9200，我们把宿主环境9200端口映射到`Docker`容器中的9200端口，就可以访问到`Docker`容器中的`ElasticSearch`服务了，同时我们把这个容器命名为`es`。

```
docker run -d --name es -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" docker.elastic.co/elasticsearch/elasticsearch:6.2.2
```



### 2.3 配置跨域

#### 2.3.1 进入容器

由于要进行配置，因此需要进入容器当中修改相应的配置信息。

```
docker exec -it es /bin/bash
```



#### 2.3.2 进行配置

```
# 显示文件
ls
结果如下：
LICENSE.txt  README.textile  config  lib   modules
NOTICE.txt   bin             data    logs  plugins

# 进入配置文件夹
cd config

# 显示文件
ls
结果如下：
elasticsearch.keystore  ingest-geoip  log4j2.properties  roles.yml  users_roles
elasticsearch.yml       jvm.options   role_mapping.yml   users

# 修改配置文件
vi elasticsearch.yml

# 加入跨域配置
http.cors.enabled: true
http.cors.allow-origin: "*"
```



### 2.4 重启容器

由于修改了配置，因此需要重启`ElasticSearch`容器。

```
docker restart es
```

访问http://IP:9200展示如下：

```
{
  "name" : "xsm9TuG",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "YEElJs8XTNux5729w6S3Vw",
  "version" : {
    "number" : "6.2.2",
    "build_hash" : "10b1edd",
    "build_date" : "2018-02-16T19:01:30.685723Z",
    "build_snapshot" : false,
    "lucene_version" : "7.2.1",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```



## 三、Docker部署ElasticSearch-Head

为什么要安装`ElasticSearch-Head`呢，原因是需要有一个管理界面进行查看`ElasticSearch`相关信息

### 3.1 拉取镜像

```
docker pull mobz/elasticsearch-head:5
```



### 3.2 运行容器

```
docker run -d --name es_admin -p 9100:9100 mobz/elasticsearch-head:5
```

展示如下：

浏览器访问：http://IP:9200/



## 四、Docker部署Kibana

### 4.1 拉取进行

```
docker pull docker.elastic.co/kibana/kibana:6.2.2
```



### 4.2 运行容器

```
docker run --name kibana --link=es:test  -p 5601:5601 -d docker.elastic.co/kibana/kibana:6.2.2
```

启动以后可以打开浏览器输入[http://IP:5601](http://localhost:5601/)就可以打开kibana的界面了。



### 4.3 解决Kibana报错：Login is currently disabled. Administrators should consult the Kibana logs for more details

### 原因：

因为Elasticesarch 安装了x-pack，并设置了密码，而kibana并未设置用户名和密码，所以kibana无法通过验证。

### 解决办法：

在kibana 的配置文件config/kibana.yml 中添加用户名和密码参数：
注意：默认用户名：elastic，密码：changeme，博主重新设置过密码为elastic

保存后，重新启动Kibana



## 五、安装ik分词器

es自带的分词器对中文分词不是很友好，所以我们下载开源的IK分词器来解决这个问题。首先进入到plugins目录中下载分词器，下载完成后然后解压，再重启es即可。具体步骤如下:
**注意：**elasticsearch的版本和ik分词器的版本需要保持一致，不然在重启的时候会失败。可以在这查看所有版本，选择合适自己版本的右键复制链接地址即可。[点击这里](https://github.com/medcl/elasticsearch-analysis-ik/releases)

进入ES

```
docker exec it es /bin/bash
```



进入到指定目录后，下载插件

```
cd /usr/share/elasticsearch/plugins/
elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.2.2/elasticsearch-analysis-ik-6.2.2.zip
exit
```



重启ES

```
docker restart elasticsearch 
```



然后可以在kibana界面的`dev tools`中验证是否安装成功；

```
POST test/_analyze
{
  "analyzer": "ik_max_word",
  "text": "你好我是东邪Jiafly"
}
```



这样，我们就完成了用Docker提供Elasticsearch服务，而不污染宿主机环境了，这样还有一个好处，如果想同时启动多个不同版本的Elastcsearch或者其他服务，Docker也是一个理想的解决方案。

