---
layout: post
title: 数据关系图可视化调研
category: ARCHIVES
---


## 1.上下文
* 视频聚类关系图展示

## 2.调研选型
* 目前实现关系图展示主要包括[14中JS展示框架](http://efe.baidu.com/blog/14-popular-data-visualization-tools/)
* 调研发现，比较匹配的包括三种，细节如下：

工具|链接|描述
---|---|---|
百度Echarts|http://echarts.baidu.com/demo.html#graph-force|效果较好、数据源支持[.gexf](https://gephi.org/gexf/format/)、json、直接输入点线信息
sigma.js|http://sigmajs.org/|效果较好、数据源支持[.gexf](https://gephi.org/gexf/format/)、json、直接输入点线信息、社区活跃度相对较弱
D3.js|https://bl.ocks.org/mbostock/4062045|效果相对差些、数据源支持[.gexf](https://gephi.org/gexf/format/)、直接输入点线信息

备注：**还没进行最大展示粒度测试**

## 3.具体实现
* 操作过程发现该问题并不只是一个前端问题，主要是后端点和线数据和点线关系的数据提供
* 后端关系图画图工具：[Gephi](https://gephi.org/)

## 4.调研结果
后面方向：

* 前端单页面最大展示粒度1万左右
* 后端关系图算法计算目前没有找到合适的计算服务工具（支持接口调用、最好是支持分布式计算）
* 需要完善展示的纬度需求和交互操作需求




