---
layout: post
title: 2023-06-25 雪花算法（附解决时钟回拨的一个超级偷懒办法）
categories: Java
description: 雪花算法（附解决时钟回拨的一个超级偷懒办法）
keywords: 雪花算法，时钟回拨
---

# 雪花算法


----------


## 简介
原名 SnowFlake，能够按照一定策略创建**有序自增**的，**分布式环境下的唯一** Id，是服务端**最常用**的 Id 生成算法。<br>
- Id 是有生成策略的，所以根据 Id 可以**反推**一些东西：
	- 数据的生成时间
	- 数据是哪台服务器生成的
- 因为是有序自增的，所以新数据插入时，将其添加到主键索引树时新节点一定在最后面。<br>
  这样就不会导致树节点的分裂和合并，也不会导致磁盘出现碎片。

## 算法逻辑

- 最高位固定值为 0，表示这个 long 型的数字是个正数
- 41 位的时间戳，时间戳的精度到毫秒级，最大值 2^41 = 2199023255552，转换成具体时间是 2039-09-07 23:47:35
- 10 位的机器码，将其划分为 5 位的 datacenterId 和 5 位的 workerId，最高支持 2^10 = 1024 台机器的分布式规模
- 12 位的序列号，当时间戳相同时，通过自增序列号来生成唯一 Id，同一台机器，同一个时间点（毫秒级）下，最多可以生成 2^12 = 4096 个 id

可以看到，算法的拓展性是很强的，完全可以自定义适合你的雪花算法。
- 减少某些部分的位数，增加到其他部分上。<br>
  比如说将机器码减少一位，将时间戳增大一位，那么时间戳的最大值 2^42 = 2199023255552，可以用到 2109-05-15 15:35:11
- 阿里的雪花算法就是标准定义的样子，没有什么变动。
- 百度的雪花算法稍微有些不同
	
### 国内大厂的雪花算法
#### 阿里 hutool-Snowflake
阿里的雪花算法就是很标准的算法，没有改什么地方。

#### 百度 uid-generator
- 最高位不变，固定为 0，表示是个正数。
- 时间戳改为**秒级**，并且缩短到 28 位，并且不是完整的时间戳，而是该时间点相对于 2016-05-20 这个时间点的增量值。<br>
  所以可以使用 2^28秒≈8.7年
- 机器码拓展到 22 位，并且进行了重定义：<br>
	标准的雪花算法，datacenterId 和 workerId 是手动配置的，所以如果真的有 1024 台服务器的话，人工去设置 datacenterId 和 workerId 是有重复的风险的。
	而百度将机器码重定义为**机器启动次数**。
	- 比如整个分布式集群中，A 机器是第一个启动，那在 A 重启之前，A 的机器码就是 1。
	- 之后 B 机器第二个启动，B 重启前，B 的机器码是 2。
	- 之后 A 重启，A 的机器码变为 3。
	- B 重启，B 的机器码变为 4。
	- ……
	- 如果是标准雪花算法，A 机器的配置文件里配置了机器码是 1，那么无论重启多少次，A 机器的机器码都是 1。
   机器码是记录在数据库中的，所以每个机器启动前，必须先去数据库中拿到最新的机器码。
- 序列号拓展到 13 位，但是**并发度降低了**，因为时间戳从毫秒级降到了秒级。<br>
  **每秒**可以生成 2^13 = 8192 个 Id
- 允许自定义时间戳、机器码、序列号的位数
- 在性能有要求的场合，可以使用 CachedUidGenerator：提前生产 Id，这样使用 Id 时直接取用即可，不需要当场生成 Id。<br>
  CachedUidGennerator 的设计还挺好的，在这里引一下：[github：百度 CachedUidGenerator](https://github.com/baidu/uid-generator/blob/master/README.zh_cn.md#cacheduidgenerator)



#### 美团 ecp-uid
基本上参考了百度 uid-generator，然后引入了 Zookeeper，Redis 等中间件来丰富机器码的获取方式。<br>
并且支持机器码的复用，而不是在数据库中单调增长。

## 时钟回拨
即服务器的时间回到了之前的时间（上一刻服务器的时间是 2023-06-25 16:27:06，下一刻却变成了 2023-06-25 16:17:06），这种事情是雪花算法来说是必须避免的。<br>
因为如果完全信任服务器的时钟，继续使用之前已经使用过的时间点来生成 Id，那么既有可能会发生 Id 冲突的事情。

### 解决方案
之前项目用的是阿里的 Snowflake，在生产中，多家医院的服务器一天内会多次发生时钟回拨（三到五分钟）的问题。

时钟回拨后，Snowflake 的策略是，在时间点推进到原来的时间之前，拒绝生成 Id，抛出异常。

比如服务器时间从 2023-06-25 17:05:24 回拨到 2023-06-25 16:55:24，那么在新的时钟从 2023-06-25 16:55:24 推进到 2023-06-25 17:05:24 之前，任何依赖于 Snowflake 的新增数据操作都会失败。

之后看了一下百度 uid-generator，发现也是抛出异常，拒绝生成 Id。

最终想了个很偷懒的办法来解决这个事。

```java
	public synchronized long nextId() {
		long timestamp = genTime();
		if (timestamp < lastTimestamp) {
			if(lastTimestamp - timestamp < 2000){
				// 容忍2秒内的回拨，避免NTP校时造成的异常
				timestamp = lastTimestamp;
			} else{
				// 如果服务器时间有问题(时钟后退) 报错。
				throw new IllegalStateException(StrUtil.format("Clock moved backwards. Refusing to generate id for {}ms", lastTimestamp - timestamp));
			}
		}

		if (timestamp == lastTimestamp) {
			sequence = (sequence + 1) & sequenceMask;
			if (sequence == 0) {
				timestamp = tilNextMillis(lastTimestamp);
			}
		} else {
			sequence = 0L;
		}

		lastTimestamp = timestamp;

		return ((timestamp - twepoch) << timestampLeftShift) | (dataCenterId << dataCenterIdShift) | (workerId << workerIdShift) | sequence;
	}
```

这是阿里 Snowflake 的源码：
- lastTimestamp 指的是上一次 Id 成功生成时的时间戳，返回 Id 前，这个值都会被写一遍。
- 新 Id 生成中，会校验当前 timestamp 和 lastTimestamp 的差值，如果回拨超过 2000ms（2000ms 内的视为可以容忍的回拨），抛出异常。

我将阿里的 cn.hutool.core.lang.Snowflake 这个类拷贝到了项目中，并且修改了 `nextId()` 这个方法
```java
    public synchronized long nextId() {
        long timestamp = genTime();

        if (timestamp < lastTimestamp) {
            timestamp = lastTimestamp;
        }

        if (timestamp == lastTimestamp) {
            sequence = (sequence + 1) & sequenceMask;
            if (sequence == 0) {
                lastTimestamp++;
                timestamp = lastTimestamp;
            }
        } else {
            lastTimestamp = timestamp;
            sequence = 0L;
        }

        return ((timestamp - twepoch) << timestampLeftShift) | (dataCenterId << dataCenterIdShift) | (workerId << workerIdShift) | sequence;
    }
```

- 如果时钟回拨了，就使用 lastTimestamp 来生成 Id
- 如果 sequence 满了，则将 lastTimestamp 往后推 1ms，即借用未来时间
- 当前 timestamp 超过 lastTimestamp 时，说明回拨结束，时间点已推进，再覆盖 lastTimestamp 和 sequence
- 然后将原来引用 hutool-Snowflake 的地方改为引用新编写的这个类