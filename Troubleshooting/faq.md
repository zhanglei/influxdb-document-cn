# FAQ
## 管理
### 如何在密码中包含单引号？
在创建密码和发送身份验证请求时使用反斜杠（\）来转义单引号。
### 如何查看InfluxDB的版本？
有几种方法可以查看InfluxDB的版本：
#### 在终端运行`influxd version`:
```
$ influxd version

InfluxDB ✨ v1.3.0 ✨ (git: master b7bb7e8359642b6e071735b50ae41f5eb343fd42)
```
#### `curl`路径`/ping`：
```
$ curl -i 'http://localhost:8086/ping'

HTTP/1.1 204 No Content
Content-Type: application/json
Request-Id: 1e08aeb6-fec0-11e6-8486-000000000000
✨ X-Influxdb-Version: 1.3.x ✨
Date: Wed, 01 Mar 2017 20:46:17 GMT
```
#### 运行InfluxDB的命令行：
```
$ influx

Connected to http://localhost:8086✨ version 1.3.x ✨  
InfluxDB shell version: 1.3.x
```
#### 在日志里查看HTTP返回结果：
```
$ journald-ctl -u influxdb.service

Mar 01 20:49:45 rk-api influxd[29560]: [httpd] 127.0.0.1 - - [01/Mar/2017:20:49:45 +0000] "POST /query?db=&epoch=ns&q=SHOW+DATABASES HTTP/1.1" 200 151 "-" ✨ "InfluxDBShell/1.3.x" ✨ 9a4371a1-fec0-11e6-84b6-000000000000 1709
```

### 怎样查看InfluxDB的日志？
在System V操作系统上，日志存储在/var/log/influxdb/下。   
在systemd操作系统上，可以使用`journalctl`访问日志。 使用`journalctl -u influxdb`查看日志或`journalctl -u influxdb> influxd.log`将日志打印到文本文件。使用systemd时，日志保留取决于系统的日记设置。

### 分片组的保留时间和保留策略之间的关系？
nfluxDB将数据存储在分片组中。一个分片组覆盖特定的时间间隔; InfluxDB通过查看相关保留策略（RP）的`DURATION`来确定时间间隔。下表列出了RP的`DURATION`和分片组的时间间隔之间的默认关系：

RP持续时间|分片组间隔
---- | ---
<2天|1小时
>= 2天，<= 6个月|1天
> 6个月|7天

用户还可以使用`CREATE RETENTION POLICY`和`ALTER RETENTION POLICY`语句配置分片组持续时间。使用`SHOW RETENTION POLICY`语句检查保留策略的分片组持续时间。

### 为什么在修改了RP之后数据没有被删除？
有几个因素可以解释为什么保留策略（RP）更改后数据可能不会立即丢失。

第一个也是最可能的原因是，默认情况下，InfluxDB每30分钟检查一次强制RP。可能需要等待下一次RP检查InfluxDB才能删除RP的新`DURATION`设置之外的数据。30分钟的间隔是可配置的。

其次，改变RP的`DURATION`和`SHARD DURATION`可能会导致意外的数据保留。InfluxDB将数据存储在包含特定RP和时间间隔的分片组中。当InfluxDB执行一个RP时，它会删除整个分片组，而不是单个数据点。InfluxDB不能拆分分片组。

如果RP的新`DURATION`小于旧的`SHARD DURATION`，并且InfluxDB正在将数据写入其中一个较旧的分片组，则系统将被迫将所有数据保留在该分片组中。即使该分片组中的某些数据不在新的`DURATION`中，也会发生这种情况。一旦所有的数据都在新的`DURATION`之外，InfluxDB将删除该分片组。系统将开始将数据写入新的分片组，这些分片组具有新的更短的`SHARD DURATION`，以防止写入不被期望的数据。

### 为什么InfluxDB无法在配置文件中解析微秒单位？
用于指定微秒持续时间单位的语法在配置设置，和写入，查询以及在InfluxDB命令行界面（CLI）中设置精度方面有所不同。 下表显示了每个类别支持的语法：

 单位|配置文件|HTTP API写入|所有的查询|CLI精度命令行
---- | ---|----|----|----
u	|❌|	👍|	👍|	👍
us|	👍|	❌|	❌|	❌
µ	|❌|	❌|	👍|	❌
µs	|👍|	❌|	❌|	❌

如果配置选项指定u或μ语法，InfluxDB将无法启动并在日志中报告以下错误：

```
run: parse config: time: unknown unit [µ|u] in duration [<integer>µ|<integer>u]
```

## 命令行