---
{"dg-publish":true,"permalink":"/01.专项学习/MySQL实战高手/02-BufferPool与内存管理/","dg-note-properties":{"时间":"2026-03-22"}}
---

#mysql #数据库 #内存管理

```ad-summary
title: 总结

- Buffer Pool 是 InnoDB 的核心内存组件，默认 128MB，生产环境要调大
- 内部三套链表：free（空闲页）、flush（脏页）、lru（淘汰顺序）
- LRU 采用冷热分离设计，避免预读和全表扫描污染热点数据
- 多实例减少锁竞争：内存 >= 1G 才能配多个实例，建议每个实例 1G 左右
- chunk 是动态扩容的最小单位（默认 128M），size 必须是 chunk_size × instances 的整数倍
- 生产环境推荐分配机器内存的 50%~60%
```

## 1. Buffer Pool 是干嘛的？

简单说就是 InnoDB 的数据缓存区。你查数据、改数据，都是先把磁盘上的页（page）加载到 Buffer Pool 里，然后在内存里操作。操作完不一定马上写回磁盘，有个后台线程定期刷盘。

所以 Buffer Pool 越大，能缓存的数据越多，磁盘 IO 就越少，性能自然好。但也不能太大，得给操作系统、其他进程、连接栈内存等留够空间。

## 2. 里面存的是什么？

存的是**数据页**，磁盘上的数据页大小是 16KB，Buffer Pool 里的缓存页跟它一一对应。

![1695276351039 1b51deb4 e2b3 434c beb7 a770162d3d9e](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/1695276351039-1b51deb4-e2b3-434c-beb7-a770162d3d9e.png)

每个缓存页都有一个**描述信息**（大概占缓存页大小的 5%-8%），记录了这个数据页属于哪个表空间、页编号是多少、在 Buffer Pool 的地址等元数据。

## 3. 三套链表

Buffer Pool 内部用三套链表管理缓存页：

### 3.1 free 链表：空闲页在哪？

用一个哈希表记录哪些缓存页是空闲的。key 是 `表空间 + 数据页号`，value 是缓存页地址。

要加载新数据页时，先查哈希表，如果没有就从磁盘加载进来。

![1695276677212 2a0f35d7 f06c 4d1f b104 ef994fca148d](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/1695276677212-2a0f35d7-f06c-4d1f-b104-ef994fca148d.png)

### 3.2 flush 链表：哪些是脏页？

你更新了 Buffer Pool 里的数据页，但还没刷回磁盘，这页就是**脏页**（跟磁盘不一致）。

flush 链表就是专门记录这些脏页的，方便后续批量刷盘。

### 3.3 lru 链表：淘汰谁？

用来管理缓存页的访问顺序，内存满了就从尾部淘汰。但 MySQL 的 LRU 不是简单的 LRU，而是**冷热分离**设计（见第 4 节）。

## 4. LRU 的冷热分离设计

普通 LRU 有个问题：预读机制或者全表扫描会把大量冷数据加载进来，把真正的热点数据挤出去。

MySQL 的解法是把 LRU 链表拆成两部分：

![1695277598145 596a1d04 af2a 4c87 9259 ace6f6d9d22b](https://raw.githubusercontent.com/Alexlindd0503/obsidian-img/main/1695277598145-596a1d04-af2a-4c87-9259-ace6f6d9d22b.png)

- **热数据区**（young 区）：63%，存放经常访问的数据
- **冷数据区**（old 区）：37%，存放新加载的数据

比例通过 `innodb_old_blocks_pct` 控制，默认 37%。

**规则**：
1. 新加载的页放在**冷数据区头部**
2. 冷数据区的页被访问后，如果**超过 1 秒**（`innodb_old_blocks_time` 默认 1000ms）还有再次访问，才移到**热数据区头部**
3. 淘汰时从**冷数据区尾部**踢

这样预读和全表扫描进来的大批量数据，如果没有后续访问，就会待在冷数据区，被淘汰时也不会影响热数据。

**热数据区的优化**：前 1/4 的缓存页被访问时不移动到头部，后 3/4 才移动，减少频繁移动带来的性能开销。

这套冷热分离的设计思路挺值得借鉴的，[[Redis内存淘汰策略清单\|Redis内存淘汰策略清单]] 里的 LRU 也是类似的思想。

## 5. 脏页什么时候刷回磁盘？

有三种触发方式：

1. **定时刷**：后台线程定期把冷数据区尾部的一些缓存页刷回磁盘，释放回 free 链表
2. **flush 链表刷**：后台线程定时把 flush 链表里的脏页刷盘，同时从 lru 链表移除，加入 free 链表
3. **紧急刷**：free 链表空了，没有空闲页了，从 lru 链表冷数据区尾部找一个缓存页强制刷盘，腾出位置

## 6. 怎么配置？

默认只有 128MB，生产环境必须调大：

```ini
[server]
innodb_buffer_pool_size = 8G
innodb_buffer_pool_instances = 4
```

### 6.1 为什么要搞多实例？

单个 Buffer Pool 实例有个问题：所有线程访问都得抢同一把锁。并发量一上来，锁竞争就成了瓶颈。

多搞几个实例，每个实例独立管理自己的内存块和链表，有自己的锁，竞争就分散了。

**注意**：总内存小于 1G 时，不管你设多少 instances，MySQL 只会给你一个实例。

### 6.2 chunk 动态扩容

Buffer Pool 不是一整块大内存，而是由很多 chunk 拼起来的，每个 chunk 默认 128M。

```mermaid
graph LR
    subgraph 扩容前["8G（4实例 × 2G）"]
        E1["实例1<br>chunk×16"]
        E2["实例2<br>chunk×16"]
        E3["实例3<br>chunk×16"]
        E4["实例4<br>chunk×16"]
    end
    
    N["申请 64 个 128M chunk"]
    
    subgraph 扩容后["16G（4实例 × 4G）"]
        E1_new["实例1<br>chunk×32"]
        E2_new["实例2<br>chunk×32"]
        E3_new["实例3<br>chunk×32"]
        E4_new["实例4<br>chunk×32"]
    end
    
    扩容前 --> N --> 扩容后
```

从 8G 扩到 16G 时，不用申请 16G 连续内存，只需要申请 64 个 128M 的小 chunk 分给各个实例，灵活得多。

### 6.3 生产环境怎么配？

**经验值**：Buffer Pool 占机器内存的 50%~60%。

**配置约束**：`innodb_buffer_pool_size` 必须是 `chunk_size × instances` 的整数倍。

**以 12G 机器、分配 8G 为例**：

```
每个实例 1G 左右 → instances = 8
chunk_size = 128M（默认）
分配单元 = 128M × 8 = 1G
8G ÷ 1G = 8，整除 ✓
```

```ini
[mysqld]
innodb_buffer_pool_size = 8G
innodb_buffer_pool_instances = 8
# chunk_size 默认 128M，一般不用动
```

**经验速查**：8G 内存配 8 个实例，16G 配 16 个，别超过 64 个。

配完可以用 [[66.归档发布/06.数据库/JDBC连接参数优化方法\|JDBC连接参数优化方法]] 里的连接池参数配合调优，让连接数和 Buffer Pool 实例数匹配，减少跨实例竞争。

## 7. 配完怎么验证？

```sql
SHOW STATUS LIKE 'Innodb_buffer_pool%';
```

重点关注：

| 指标 | 说明 |
|------|------|
| `Innodb_buffer_pool_pages_total` | 总页数，× 16K 就是实际分配的内存 |
| `Innodb_buffer_pool_pages_free` | 空闲页数，长期很少说明 BP 偏小 |
| `Innodb_buffer_pool_read_requests` | 逻辑读次数 |
| `Innodb_buffer_pool_reads` | 物理读次数（缓存未命中） |

**缓存命中率** = 1 - (物理读 / 逻辑读)，一般要 99% 以上。