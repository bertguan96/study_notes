# Elasticsearch

## 数据读取

通过docId Hash找到数据备份到哪一片上，然后再去那片进行查询。

具体流程：

- 客户端发送请求到任意一个 node，成为 coordinate node（负责查询数据的准备，并转发对应的查询请求）。
- coordinate node 对 doc id 进行哈希路由，将请求转发到对应的 node，此时会使用 round-robin随机轮询算法（为了确保**负载均衡**），在 primary shard以及其所有 replica 中随机选择一个，让读请求负载均衡。
- 接收请求的 node 返回 document 给 coordinate node。
- coordinate node 返回 document 给客户端。

## 数据搜索

- 客户端发送请求到一个 coordinate node。
- 协调节点将搜索请求转发到所有的 shard 对应的 primary shard 或 replica shard，都可以。
- query phase：每个 shard 将自己的搜索结果（其实就是一些 doc id）返回给协调节点，由协调节点进行数据的合并、排序、分页等操作，产出最终结果。
- fetch phase：接着由协调节点根据 doc id 去各个节点上拉取实际的 document数据，最终返回给客户端。

**注意**

写请求是写入 primary shard，然后同步给所有的 replica shard；

读请求可以从 primary shard 或 replica shard 读取，采用的是随机轮询算法；

## 写入数据

1.先写入内存 buffer，同时将数据写入 translog 日志文件，在 buffer 里的时候数据是搜索不到的；

2.如果 buffer 快满了，或者到一定时间，就会将内存 buffer 数据refresh到一个新的segment file（1s一次）里面；

3.此时数据不是直接进入segment file的会先进入os cache，等到一定时间后刷新到segment file中去，当数据到了os cache这一步就可以被搜索到了。

## 准实时

默认是每隔 1 秒 refresh 一次的，所以 es 是准实时的，因为写入的数据 1 秒之后才能被看到。如果想提前看到，可以调用Java API或者restful api提前刷进去（这里是刷到os cache中）。

## 灾备

当我们操作ES的时候，数据不是存储在buffer中就是在os cache中，假如此时ES发生了崩溃，内存数据就会全丢了，所以就需要一个专门的日志文件用来恢复，即为：translog。

一旦此时机器宕机，再次重启的时候，es 会自动读取 translog 日志文件中的数据，恢复到内存 buffer 和 os cache 中去。

**translog 其实也是先写入 os cache 的，默认每隔 5 秒刷一次到磁盘中去，**所以默认情况下，可能有 5 秒的数据会仅仅停留在 buffer 或者 translog 文件的 os cache 中，如果此时机器挂了，会丢失 5 秒钟的数据。

## 删除数据与更新

如果是删除操作，commit 的时候会生成一个 .del 文件，里面将某个 doc 标识为 deleted状态，那么搜索的时候根据 .del文件就知道这个 doc 是否被删除了。

如果是更新操作，就是将原来的 doc 标识为 deleted 状态，然后新写入一条数据。

**buffer 每 refresh 一次，就会产生一个 segment file，所以默认情况下是 1 秒钟一个 segment file，**这样下来 segment file 会越来越多，此时会定期执行 merge。每次 merge 的时候，会将多个 segment file 合并成一个，同时这里会将标识为 deleted 的 doc 给物理删除掉，然后将新的 segment file 写入磁盘，这里会写一个 commit point，标识所有新的 segment file，然后打开 segment file 供搜索使用，同时删除旧的 segment file。

## lucene

// todo

## 倒排索引

// todo

## 参考文献

https://pdai.tech/md/db/nosql-es/elasticsearch-y-th-4.html#%E5%8D%95%E4%B8%AA%E6%96%87%E6%A1%A3

https://juejin.cn/post/6914570948427874317#heading-2