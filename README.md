# 介绍

一致性哈希环(circle)，一个 circle 包含了若干个 influxdb 实例，共同存储了一份全量的数据，即每个 circle 都是全量数据的一个副本，各个 circle 数据互备。

一个 circle 是一个逻辑上的一致性哈希环，包含少数的物理节点（其中物理节点就是实际的influxDB实例）和更多数的虚拟节点

每个 circle 维护了一份全量数据，一个 influxdb 实例上的数据只是从属 circle 数据的一部分

每个 circle 数据存储位置计算：

`shard_key(db,measurement) + hash_key(idx) + influxdb实例列表 + 一致性哈希算法 => influxdb实例`

> 当 influxdb实例列表、hash_key、shard_key 不发生改变时，db,measurement 将只会唯一对应一台 influxdb实例（也就是db+measurement唯一对应一个influxdb实例）

## 写入数据流程
当influx-proxy接收到请求的时候，influx-proxy下的每个circle都会根据请求中的db+measurement来唯一确定一个influxdb实例，然后把请求转发给这个influxdb实例，由这个实例执行write操作。一旦所有选定的InfluxDB实例确认写入成功，influx-proxy将向客户端返回写入成功的响应。


> 如果这个唯一确定的实例宕机了，influx-proxy 会将数据写入到缓存文件中，并直到 influxdb 实例恢复后重新写入

## 查询数据流程
和写入数据同理，会根据当influx-proxy接收到查询请求的时候，influx-proxy下的每个 circle 会根据请求中的db+measurement来唯一确定一个influxdb实例，此时在这些influxdb实例中，随机挑选一个健康的实例，然后把请求转发给这个influxdb实例，由这个实例执行query操作。

但如果请求中只有db，没有measurement，那么就会视为一个全集群的查询语句，此时会将请求转发给该 circle 的所有 influxdb 实例。

最终，由 influxdb 实例处理查询请求，读出数据，然后返回给 influx-proxy，再返回给客户端。

> 若是单个实例返回数据（即db+measurement的情况），则直接返回结果给客户端；
> 若是多个实例返回数据（即只有db的情况），则返回给 influx-proxy 合并，然后再返回给客户端。


## 举例
假如现在集群的部署情况是：1个influx-proxy实例，influx-proxy实例下有3个circle，每个circle有3个influxdb实例。

当来了一个写请求的时候：influx-proxy下的每个circle都会根据db+measurement来计算各自circle中的influxdb实例，假如此时计算出来是circle-1中的influxdb-1-1，circle-2中的influxdb-2-1，circle-3中的influxdb-3-1这三个实例，那么这3个实例会把数据写入自己的实例中。

当来了一个查询请求的时候：和写请求同理，假如此时计算出来是circle-1中的influxdb-1-2，circle-2中的influxdb-2-2，circle-3中的influxdb-3-2这三个实例，那么会随机选择1个健康的实例，然后把数据读出来，返回给客户端。



# 使用
## 部署集群
```bash
docker-compose up -d
```

## 集群写数据
```bash
curl --request POST "http://localhost:8080/api/v2/write?org=klec-org&bucket=klec-bucket&precision=s"  --header "Authorization: Token 123456" --header "Content-Type: text/plain; charset=utf-8" --header "Accept: application/json" --data-binary "home,room=Living temp=21.1,hum=35.9,co=0i 1729128244"
```

## 集群查询数据
```bash
curl --request POST "http://localhost:8080/api/v2/query?org=klec-org&bucket=klec-bucket" --header "Authorization: Token 123456" --header "Content-Type: application/vnd.flux" --header "Accept: application/csv" --data 'from(bucket: "klec-bucket") |> range(start: 0) |> filter(fn: (r) => r._measurement == "home")'
```

## 查看数据分片
使用`docker exec -it influxdb-1 bash`进入容器，使用`influx query 'from(bucket: "klec-bucket") |> range(start: 0)'`查询所有数据

