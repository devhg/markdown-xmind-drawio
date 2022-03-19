redis参数调优 https://blog.csdn.net/a17816876003/article/details/114150748



![netty-1](https://image-ihui.oss-cn-beijing.aliyuncs.com/img/netty-1.png)



## 测试0

### 100连接，pub，每个连接100条200byte

```
$ mqtt-benchmark --broker tcp://127.0.0.1:1883 --count 100 --size 200 --clients 100 --qos 2 -topic mqtt-bench --format text
========= TOTAL (100) =========
Total Ratio:                 1.000 (10000/10000)
Total Runtime (sec):         101.256
Average Runtime (sec):       101.128
Msg time min (ms):           3.863
Msg time max (ms):           176.574
Msg time mean mean (ms):     17.217
Msg time mean std (ms):      2.761
Average Bandwidth (msg/sec): 0.989
Total Bandwidth (msg/sec):   98.885
```



## 测试1：500sub 500pub

### Sub 500 连接 50000条消息

```
root@debian:~/GoWork/src/github.com/devhg/mqtt-mock# ./mqtt-mock -broker "tcp://192.168.2.163:1883" -c 500 -n 50000 -topic mqtt-mock/benchmark -qos 1 -action sub
Mock Info:
	broker:       tcp://192.168.2.163:1883
	c:            500
	n:            50000
	username:     admin
	password:     123456
	topic:        mqtt-mock/benchmark
	qos:          1
	debug:        false
2022/03/15 16:35:11 Throughput=62.50(messages/sec)
2022/03/15 16:35:13 Throughput=107.00(messages/sec)
2022/03/15 16:35:15 Throughput=2601.00(messages/sec)
2022/03/15 16:35:17 Throughput=1202.50(messages/sec)
2022/03/15 16:35:19 Throughput=96.50(messages/sec)
2022/03/15 16:35:21 Throughput=104.50(messages/sec)
2022/03/15 16:35:23 Throughput=106.00(messages/sec)
2022/03/15 16:35:25 Throughput=91.50(messages/sec)
2022/03/15 16:35:27 Throughput=104.50(messages/sec)
2022/03/15 16:35:29 Throughput=89.00(messages/sec)
2022/03/15 16:35:31 Throughput=96.00(messages/sec)
2022/03/15 16:35:33 Throughput=92.50(messages/sec)
2022/03/15 16:35:35 Throughput=84.00(messages/sec)
2022/03/15 16:35:37 Throughput=3653.50(messages/sec)
2022/03/15 16:35:39 Throughput=99.00(messages/sec)
2022/03/15 16:35:41 Throughput=105.50(messages/sec)
2022/03/15 16:35:43 Throughput=7945.50(messages/sec)
2022/03/15 16:35:45 Throughput=2663.00(messages/sec)
2022/03/15 16:35:47 Throughput=2310.50(messages/sec)
2022/03/15 16:35:49 Throughput=46.50(messages/sec)
2022/03/15 16:35:51 Throughput=1756.00(messages/sec)
2022/03/15 16:35:51 Finish subscribe mock! total=50000 cost=42s Throughput=1190.48(messages/sec)
```



### Pub 500连接，每个连接100个200byte的消息

```
mqtt-benchmark --broker tcp://127.0.0.1:1883 --count 100 --size 200 --clients 500 --qos 0 -topic mqtt-mock/benchmark --format text
```

