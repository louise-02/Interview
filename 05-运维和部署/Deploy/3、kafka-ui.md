# docker 安装

> 公共步骤见 [docker/1、环境准备.md](./docker/1、环境准备.md)

## 1、Docker Compose 启动（推荐，与 Kafka 一起）

Kafka UI 已包含在 [docker/zookeeper-kafka.yml](./docker/zookeeper-kafka.yml) 中，启动后访问 `http://宿主机IP:8080`。

```bash
cd /path/to/Deploy/docker
# 修改 zookeeper-kafka.yml 中 KAFKA_CFG_ADVERTISED_LISTENERS 的宿主机IP
docker compose -f zookeeper-kafka.yml up -d
```

## 2、单独连接已有集群

```bash
docker run -d \
  --name kafka-ui \
  -p 8080:8080 \
  -e KAFKA_CLUSTERS_0_NAME=dev-cluster \
  -e KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=10.168.106.107:9092,10.168.106.108:9092,10.168.106.109:9092 \
  -e KAFKA_CLUSTERS_0_ZOOKEEPER=10.168.106.107:2181,10.168.106.108:2181,10.168.106.109:2181 \
  provectuslabs/kafka-ui
```
