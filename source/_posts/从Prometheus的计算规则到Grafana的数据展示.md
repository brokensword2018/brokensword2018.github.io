---
title: 从Prometheus的计算规则到Grafana的数据展示
date: 2020-08-16 17:02:33
tags:
    - 普罗米修斯
    - Grafana
  
categories: 工具学习
---

## 1. 前序  
本文目标为阐释清楚：**一条`PromQL`在执行过程中发生了什么，从而得到`Grafana`中的图像**
<!-- more -->

## 2. `PromQL`所操作的对象 
首先明确
1. `PromQL`计算依赖于收集数据的`timestamp`
2. 收集数据的`timestamp`是`Prometheus server`发出`GET`请求时的时间戳
3. `Prometheus server`发出`GET`请求的时间并不严格符合数据的收集间隔
   
ps: `timestamp`函数可以查看收集上来数据的时间戳
接下来对上述第3点做简单解释。

{% asset_img Prometheus的GET请求时间.jpg Prometheus的GET请求时间 %}

如上图所示，蓝色的点为按照我们设置的数据收集间隔，`GET`请求的理论时间，理论时间间隔严格等于设置的收集时间间隔。然而实际上当`Prometheus server`负载较大时，实际发出`GET`请求的时间相对于理论时间可能有一定的延迟，如`a)`所示; 当`Prometheus server`同时向多个`Instance`收集数据时，`Prometheus server`甚至可能将多个`GET`请求离散分布到收集数据的时间间隔中，如`b)`所示。

##  3. `Instant/Range queries`和`Instant/Range vector selectors`
[Instant/Range vector selectors](https://prometheus.io/docs/prometheus/latest/querying/basics/)是两种查询语句。
- `Instant vector selectors`：在给定时间处从时间序列中选择出一个样本点
- `Range vector selectors`：从时间序列中选出`timestamp`在区间`[给定时间 - range duration, 给定时间]`的样本点
  
[Instant/Range queries](https://prometheus.io/docs/prometheus/latest/querying/api/)是查询时间的范围的区别。
- `Instant queries`：对某一个时间点进行查询。
- `Range queries`：对某一段时间区间进行查询。`step`参数是指将查询区间按照`step`大小的间隔划分得到多个时间点，在这些时间点上使用`Instant/Range vector selectors`。可以这样理解，`Range queries`通过`step`参数转化为了多个`Instant queries`。假设查询区间为`[0, 90]s`,`step`参数为30s,那么该查询就变成了在`0s, 30s, 60s, 90s`的四次查询。在Grafana中显示横坐标就是时间，纵坐标就是值。Grafana显示变化趋势的图像一定是`Range queries`。

  
假设一个`counter`类型的统计数据，名字为`http_request`,它的统计结果如下图所示：
{% asset_img http_request假设数据.bmp http_request假设数据 %}

### 3.1. `Instant queries` + `Instant vector selectors`
- 假定查询时间为`50s`
- 查询语句直接为`http_request`
查询结果为15，如下图所示。`Instant vector selectors`的查询规则为**向后搜索到第一个时间戳小于等下查询时间的样本点；若搜索距离超过5分钟仍然无样本点，则查询结果为空**。
{% asset_img instant_queries和instant_vector_selector.bmp instant_queries和instant_vector_selector %}

### 3.2. `Instant queries` + `Range vector selectors`
- 假定查询时间为65s
- 查询语句为`http_request[1m]`
查询结果为`7(15s), 9(30s), 15(45s), 17(60s)`。我们可以使用一些函数作用于这些查询结果,例如`increase`函数，结果为10。表明查询时间处的`qpm`为10。示意图如下

{% asset_img instant_queries和range_vector_selector.bmp instant_queries和range_vector_selector %}

### 3.3. `Range queries` + `Instant vector selectors`
- 查询区间为`[0, 90]s`,`step`参数为30s
- 查询语句直接为`http_request`
查询结果为`1(0s), 9(30s), 17(60s), 30(90s)`。如下图所示
{% asset_img range_queries和instant_vector_selector.bmp range_queries和instant_vector_selector %}

### 3.4. `Range queries` + `Range vector selectors`
- 查询区间为`[0, 90]s`,`step`参数为30s
- 查询语句直接为`http_request[15s]`
查询结果为`[1(0s)], [7(15s),9(30s)], [15(45s),17(60s)], [23(75s),30(90s)]`如下图所示。`Range vector selectors`的计算结果无法直接用于作图，需要使用函数对其进行计算才能进行图像绘制。例如使用`increase`函数，结果为`空, 2, 2, 7`，空是因为第一组数据只有一个数`increase`函数无法计算。
{% asset_img range_queries和range_vector_selector.bmp range_queries和range_vector_selector %}

## 4. `Grafana`中的图像展示  
Grafana中Prometheus的参数如下所示，其中Interval 乘以 Resolution（显示精度）就是 Prometheus参数。
{% asset_img Grafana中Prometheus参数.bmp Grafana中Prometheus参数 %}
ps: 可通过Query inspector查看具体的step


## 5. 参考
[Prometheus官方文档](https://prometheus.io/docs/introduction/overview/)
[Grafana文档](https://grafana.com/docs/grafana/latest/)
[详解Prometheus range query中的step参数](https://segmentfault.com/a/1190000017553625)