### 项目启动

````
先进入 WSL Ubuntu，然后进入项目的 Docker 目录：

```
cd ~/edgelite-lab/EdgeLiteGateway/docker
```

启动整个项目：

```
sudo docker compose up -d
```

这会启动目前的三个核心容器：

```
edgelite-gateway
├─ EdgeLite 后端
├─ Simulator
├─ Scheduler
└─ 数据预处理

edgelite-influxdb
└─ 时序数据库

edgelite-mosquitto
└─ Docker 内 MQTT Broker
```

然后检查状态：

```
sudo docker compose ps
```

正常应该类似：

```
edgelite-gateway      Up ... (healthy)
edgelite-influxdb     Up ... (healthy)
edgelite-mosquitto    Up ... (healthy)
```

也可以检查 Gateway：

```
curl -s http://127.0.0.1:8080/health/live
```

正常：

```
{"status":"ok"}
```

最后浏览器打开：

```
http://localhost:8080
```

即可进入 EdgeLite 页面。

````

 VS Code 看源码

```
cd ~/edgelite-lab/EdgeLiteGateway

code .
```

```
# 进入 Docker 目录

cd ~/edgelite-lab/EdgeLiteGateway/docker

# 启动

sudo docker compose up -d

# 查看状态

sudo docker compose ps

# 查看 Gateway 日志

sudo docker logs --tail 50 edgelite-gateway

# 实时查看 Gateway 日志

sudo docker logs -f edgelite-gateway

# 重启 Gateway

sudo docker compose restart edgelite

# 停止项目

sudo docker compose stop

# 再次启动已经存在的容器

sudo docker compose start
```

### Moving avg

#### raw+noise

![image-20260916191628474](C:\Users\Zuu\AppData\Roaming\Typora\typora-user-images\image-20260916191628474.png)

#### Moving avg

![image-20260916191727273](C:\Users\Zuu\AppData\Roaming\Typora\typora-user-images\image-20260916191727273.png)

**当前时刻的输出，不直接使用当前这一个采样值，而是取最近 N 个采样值的平均值。** 

假设窗口大小 `N=5`，采样序列是：

```
x1, x2, x3, x4, x5, x6, x7 ...
```

那么从窗口装满之后：

```
y5 = (x1+x2+x3+x4+x5) / 5
y6 = (x2+x3+x4+x5+x6) / 5
y7 = (x3+x4+x5+x6+x7) / 5
```

窗口不断向前移动，所以叫“滑动平均”。

```
滑动平均没有明显改变温度信号的总体趋势和平均水平，但有效抑制了高频随机噪声以及极端瞬时波动。
```

### Median

#### Raw Spike

![image-20260916204255817](C:\Users\Zuu\AppData\Roaming\Typora\typora-user-images\image-20260916204255817.png)

#### Median

![image-20260916205641852](C:\Users\Zuu\AppData\Roaming\Typora\typora-user-images\image-20260916205641852.png)

```
保存最近 N 个采样点，排序后取中间那个值作为输出，用来消除孤立尖峰。

原始数据：

50.1   50.5   80.0   50.9   51.2
              ↑
            异常尖峰

窗口：
[50.5, 80.0, 50.9]

排序：
[50.5, 50.9, 80.0]

中值：
50.9

80.0
 ↓ Median(3)
50.9

中值滤波维护最近 N 个采样点的滑动窗口，将窗口数据排序并取中位数作为输出，能够有效抑制孤立脉冲和异常尖峰。
```

### Ring Buffer（环形缓冲区)

适用范围：保存“最近 N 个数据”的实时场景。
 实现方式：固定数组 + 写指针循环覆盖最旧数据。

