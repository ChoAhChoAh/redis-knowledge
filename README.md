# redis-knowledge
基于《redis设计与实现》，结合先redis最新源码（unstable分支，截至2024年4月），记录redis底层知识的学习。
redis源码分类
1.服务器相关
服务器实例入口：server.h/server.c
基于事件驱动的网络通讯框架：ae.h/ae.c,ae_epoll.c,ae_evport.c,ae_kqueue.c,ae_select.c
底层tcp网络通讯实现：anet.h/anet.c
客户端实现：networking

2.redis内存数据库操作
数据库操作：db.c
sds(String)：sds.h/sds.c/sdsalloc.h
双向链表(List)：adlist.h/adlist.c
压缩列表(List、Hash、Sorted Set)：ziplist.h/ziplist.c
QuickList(List、Hash、Sorted Set)：quicklist.h/quicklist.c
整数列表(Set)：intset.h/intset.c
zipmap(Hash)：zipmap.h/zipmap.c
字典（hash）：dict.h/dict.c
HyperLogLog(HyperLogLog)：hyperloglog.c
GeoHash(Geo)：geo.h/geo.c,geohash.h/geohash.c,geohash_helper.h/geohash_helper.c
位图(位图)：bitops.c
Stream（时序数据）：stream.h/t_stream.c
内存分配：zmalloc.c/zmalloc.h
内存回收：lazyfree.c
数据替换：evict.c

3.高可靠性和高可扩展性
数据持久化实现：rdb.h/rdb.c，aof.c
文件检查文件：redis-check-rdb.c/redis-check-aof.c
主从复制功能实现：replication.c
高可扩展：cluster.h/cluster.c

4.辅助功能
操作延迟监控：latency.h/latency.c
慢命令记录：showlog.h/showlog.c
性能评测：redis-benchmark.c