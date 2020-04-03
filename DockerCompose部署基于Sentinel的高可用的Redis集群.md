# Docker Compose部署基于Sentinel的高可用Redis集群

[TOC]

Redis集群可以在一组redis节点之间实现高可用性和sharding。今天我们重点围绕master-slave的高可用模式来进行讨论，在集群中会有1个master和多个slave节点。当master节点失效时，应选举出一个slave节点作为新的master。然而Redis本身(包括它的很多客户端)没有实现自动故障发现并进行主备切换的能力，需要外部的监控方案来实现自动故障恢复。

Redis Sentinel是官方推荐的高可用性解决方案。它是Redis集群的监控管理工具，可以提供节点监控、通知、自动故障恢复和客户端配置发现服务。

本文采用的Redis镜像全部基于Docker提供的[Redis官方镜像3.2.1](https://yq.aliyun.com/go/articleRenderRedirect?spm=a2c4e.11153940.0.0.1f1d3aa8Vb5Qo4&url=https%3A%2F%2Fhub.docker.com%2F_%2Fredis%2F)

## 单机部署Redis集群

下面的测试需要本地环境已经安装Docker Engine和Docker Compose，推荐使用Docker for Mac/Windows。

docker-compose.yml模板定义Redis集群的服务组成

```
master:
  image: redis:3
slave:
  image: redis:3
  command: redis-server --slaveof redis-master 6379
  links:
    - master:redis-master
sentinel:
  build: sentinel
  environment:
    - SENTINEL_DOWN_AFTER=5000
    - SENTINEL_FAILOVER=5000    
  links:
    - master:redis-master
    - slave
```

在模板中定义了下面一系列服务

- master: Redis Master
- slave: Redis Slave
- sentinel: Redis Sentinel

其中sentinel服务的Docker镜像是由 "./sentinel" 目录中的Dockerfile构建完成，只是在官方Redis镜像上添加了`sentinel.conf`配置文件，并以sentinel模式启动容器。其配置文件如下，其中包含了sentinel对名为"mymaster"的集群的监控配置：

```
sentinel monitor mymaster redis-master 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 5000
```

**注意：**

- slave和sentinel容器初始化配置的RedisMaster节点主机名为“redis-master”，这里我们利用了Docker容器链接的别名机制来链接master和sentinel/slave容器实例
- 由于要部署3个Sentinel，我们把sentinel的“quorum”设置为2，只有两个sentinel同意故障切换，才会真正切换相应的redis master节点。

下面先构建sentinel服务所需要Docker image

```
docker-compose build
```

一键部署并启动Redis集群

```
docker-compose up -d
```

可以检查集群状态，应该是包含3个容器，1个master，1个slave，和1个sentinel

```
docker-compose ps
```

显示结果如下：

```
/opt/docker/redis-cluster/sentinel # docker-compose ps
Name                        Command               State          Ports       
---------------------------------------------------------------------------------------
redis-cluster_master_1     docker-entrypoint.sh redis ...   Up      6379/tcp           
redis-cluster_sentinel_1   sentinel-entrypoint.sh           Up      26379/tcp, 6379/tcp
redis-cluster_slave_1      docker-entrypoint.sh redis ...   Up      6379/tcp
```

可以所sentinel的实例数量到3个

```
docker-compose scale sentinel=3
```

伸缩slave的实例数量到2个，这样我们就有3个redis实例了（包含1个master）

```
docker-compose scale slave=2
```

检查集群状态，结果如下：

```
/opt/docker/redis-cluster/sentinel # docker-compose ps
Name                        Command               State          Ports       
---------------------------------------------------------------------------------------
redis-cluster_master_1     docker-entrypoint.sh redis ...   Up      6379/tcp           
redis-cluster_sentinel_1   sentinel-entrypoint.sh           Up      26379/tcp, 6379/tcp
redis-cluster_sentinel_2   sentinel-entrypoint.sh           Up      26379/tcp, 6379/tcp
redis-cluster_sentinel_3   sentinel-entrypoint.sh           Up      26379/tcp, 6379/tcp
redis-cluster_slave_1      docker-entrypoint.sh redis ...   Up      6379/tcp           
redis-cluster_slave_2      docker-entrypoint.sh redis ...   Up      6379/tcp
```

可以利用下面的测试脚本来模拟master节点失效，并验证Redis集群的自动主从切换：

```
./test.sh
```

这个测试脚本实际上是利用docker pause命令将Redis master容器暂停，sentinel会发现这个故障并将master切换到其他一个备用的slave上面，执行结果如下：

```
/opt/docker/redis-cluster # ./test.sh
Redis master: 172.17.0.4
Redis Slave: 172.17.0.5
------------------------------------------------
Initial status of sentinel
------------------------------------------------
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=172.17.0.4:6379,slaves=3,sentinels=3
Current master is
172.17.0.4
6379
------------------------------------------------
Stop redis master
redis-cluster_master_1
Wait for 10 seconds
Current infomation of sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=172.17.0.5:6379,slaves=3,sentinels=3
Current master is
172.17.0.5
6379
------------------------------------------------
Restart Redis master
redis-cluster_master_1
Current infomation of sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=172.17.0.5:6379,slaves=3,sentinels=3
Current master is
172.17.0.5
6379
```

我们可以利用Docker Compose方便地在本地验证Redis集群的部署和故障恢复，但是这还不是一个分布式的高可用部署。

**总结**

出于性能考虑，在Docker容器中运行Redis不建议采用bridge网络对外提供访问，如需对外部VM或应用提供服务建议采用host网络模式，并注意安全保护；如果只是对集群中容器提供redis访问，则容器服务默认提供的跨宿主机容器网络会提供优化而安全的网络配置。同时建议在Docker容器设置中，给Redis容器配置合适的内存设置。