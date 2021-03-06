# [GEOADD](https://redis.io/commands/geoadd)

GEOADD key longitude latitude member [longitude latitude member ...]

```
可用版本： >=3.2.0
时间复杂度： 每添加一个元素的复杂度为 O(log(N))，其中 N 为键里面包含的位置元素数量。
```

将指定的地理位置元素（经度、纬度、名称）添加到指定的键中。这些数据会以有序集合的形式被储存在键里面，从而使得像 ``GEORADIUS`` 和 ``GEORADIUSBYMEMBER`` 这样的命令可以在之后通过位置查询取得这些元素。

``GEOADD`` 命令以标准的 ``x,y`` 格式接受参数，所以经度必须在纬度之前。``GEOADD`` 能够记录的坐标是有限的：非常接近两极的区域是无法被记录的。精确的坐标限制由 EPSG:900913 / EPSG:3785 / OSGEO:41001 等坐标系统定义，具体如下：

* 有效的经度介于 -180 度至 180 度之间。
* 有效的纬度介于 -85.05112878 度至 85.05112878 度之间。

当用户尝试输入一个超出范围的经度或者纬度时，``GEOADD`` 命令将返回一个错误。

**注解**： Geo 地理位置坐标存储在有序集合，所以没有 ``GEODEL``/``GEODEL`` 这种删除命令，因为可以使用 ``ZREM`` 进行删除。

## 工作原理

地理位置信息通过 GeoHash 技术方案存储到有序集合中，经纬度经过转换生成唯一的 52 位整数，因为有序集合的 Score 为双精度浮点数，可以存储 52 位整数，而不丢失精度。

```
float：
  1bit（符号位） 8bits（指数位） 23bits（尾数位）
double：
  1bit（符号位） 11bits（指数位） 52bits（尾数位）
```

2 ** 53 == 9007199254740992

```shell
127.0.0.1:6379> ZADD precision 9007199254740992 'max'
(integer) 1
127.0.0.1:6379> ZRANGE precision 0 -1 WITHSCORES
1) "max"
2) "9007199254740992"
127.0.0.1:6379> ZADD precision 9007199254740993 'max'
(integer) 0
127.0.0.1:6379> ZRANGE precision 0 -1 WITHSCORES
1) "max"
2) "9007199254740992"
```

This format allows for radius querying by checking the 1+8 areas needed to cover the whole radius, and discarding elements outside the radius. The areas are checked by calculating the range of the box covered removing enough bits from the less significant part of the sorted set score, and computing the score range to query in the sorted set for each area.

## 返回值

新添加到有序集合里的元素数量， 不包括那些已存在被更新的元素。

## 示例

```shell
$ redis-cli

# 添加地理位置坐标

127.0.0.1:6379> GEOADD jscities 0 0 nanjing
(integer) 1  -- 添加地理位置坐标
127.0.0.1:6379> ZRANGE jscities 0 -1 WITHSCORES
1) "nanjing"  -- 存储方式为 Sorted Sets， 返回存储的 Member
2) "3377699720527872"  -- 存储方式为　Sorted Sets， 返回存储的 GeoHash 值
127.0.0.1:6379> GEORADIUS jscities 0 0 +inf km WITHDIST WITHCOORD WITHHASH
1) 1) "nanjing"  -- Member
   2) "0.0003"  -- 与 (0, 0) 的距离， 单位千米
   3) (integer) 3377699720527872  -- GeoHash 值
   4) 1) "0.00000268220901489"  -- 精度
      2) "0.00000126736058093"  -- 纬度

# 修改地理位置坐标

127.0.0.1:6379> GEOADD jscities 118.7964667635 32.0583898428 nanjing
(integer) 0  -- 修改地理位置坐标
127.0.0.1:6379> GEORADIUS jscities 0 0 +inf km WITHDIST WITHCOORD WITHHASH
1) 1) "nanjing"  -- Member
   2) "12690.3180"  -- 与 (0, 0) 的距离， 单位千米
   3) (integer) 4066006847826700  -- GeoHash 值
   4) 1) "118.7964668869972229"  -- 精度
      2) "32.05838900488951282"  -- 纬度

# 添加多个地理位置坐标

127.0.0.1:6379> GEOADD jscities 117.2857487834 34.2043945878 xuzhou 120.5831901464 31.2983274306 suzhou 120.3123694224 31.4909547345 wuxi
(integer) 3  -- 添加多个地理位置坐标
127.0.0.1:6379> GEORADIUS jscities 0 0 +inf km WITHDIST WITHCOORD WITHHASH
1) 1) "suzhou"
   2) "12876.5781"
   3) (integer) 4054417970294782
   4) 1) "120.58318823575973511"
      2) "31.29832751805013658"
2) 1) "wuxi"
   2) "12845.7197"
   3) (integer) 4054420329652683
   4) 1) "120.31237095594406128"
      2) "31.49095365255409718"
3) 1) "xuzhou"
   2) "12488.5192"
   3) (integer) 4064842258364436
   4) 1) "117.28575021028518677"
      2) "34.20439546611687831"
4) 1) "nanjing"
   2) "12690.3180"
   3) (integer) 4066006847826700
   4) 1) "118.7964668869972229"
      2) "32.05838900488951282"
```
