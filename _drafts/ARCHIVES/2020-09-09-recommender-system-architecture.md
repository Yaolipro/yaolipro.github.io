推荐系统微服务架构

Gateway：调度和聚合其他下游业务模块，读取用户信息(兴趣、曝光)

Retrieve：负责召回，并进行粗排序，支持模型参数实时更新，实时计算召回结果，同时支持获取离线算好的结果

Ranker：推荐排序模块，mml-serving／默认排序

Reranker：策略模块，打散等人工策略

Index：索引服务，提供文本、检索功能

FeedCache：缓存服务，缓存用户的召回结果(不必每次实时计算，提供系统性能)

Db：注意是Redis、Pika，存储推荐、用户画像、曝光结果等

Impserver：分布式曝光拍重服务



推荐服务流程：

1. app请求推荐系统
2. gateway获取用户兴趣
3. gateway把用户信息传递给retrieve，retrieve按照配置多路进行召回
4. 根据产品位获取配置参数，调用index召回服务活着直接获取离线计算结果
5. gateway把retrieve返回的结果取topN，传到ranker进行排序
6. reranker进行再排序
7. gateway把推荐结果存入曝光服务
8. 返回给前端