# 任务总览

实现数据库索引。
实现B+Tree动态索引结构，动态增长收缩，需要实现分离合并逻辑。

分为四个任务。
依赖于之前实现的缓存池。

[官方可视化参考](https://15445.courses.cs.cmu.edu/fall2022/bpt-printer/)
[一个b+树实现例程](https://github.com/sayef/bplus-tree)

# 配置

```c++ b_plus_tree_internal_page.h
static constexpr int BUSTUB_PAGE_SIZE = 4096;  // size of a data page in byte
define INTERNAL_PAGE_HEADER_SIZE 24
define INTERNAL_PAGE_SIZE ((BUSTUB_PAGE_SIZE - INTERNAL_PAGE_HEADER_SIZE) / (sizeof(MappingType)))

 * Store n indexed keys and n+1 child pointers (page_id) within internal page.
 * Pointer PAGE_ID(i) points to a subtree in which all keys K satisfy:
 * K(i) <= K < K(i+1).
 * NOTE: since the number of keys does not equal to number of child pointers,
 * the first key always remains invalid. That is to say, any search/lookup
 * should ignore the first key.
 * Internal page format (keys are stored in increasing order):
 *  --------------------------------------------------------------------------
 * | HEADER | KEY(1)+PAGE_ID(1) | KEY(2)+PAGE_ID(2) | ... | KEY(n)+PAGE_ID(n) |
 *  --------------------------------------------------------------------------

```
# 参考博客

https://zhuanlan.zhihu.com/p/592964493
[十一哥的博客：写的非常全面，非常好](https://blog.eleven.wiki/posts/cmu15-445-project2-b+tree-index/)
https://www.cnblogs.com/alyjay/p/16885958.html
https://blog.csdn.net/Altair_alpha/article/details/129071063
https://www.cnblogs.com/sun-lingyu/p/15198683.html

# 理解

![](Pasted%20image%2020230513172936.png)
下图是缓存池中的一个页Page类型：除了数据信息还有元数据信息。
![](Pasted%20image%2020230514135144.png)
而==我们实现的B+Tree 页（节点）为Page.data_，一个节点大小为4k。不包括元数据信息。==
**每个b+树叶/内部节点页对应的页内容（data_）由缓存池获取，每次读写树页需要先从缓存池中使用`page_id`取页，然后使用`reinterpret cast`强转，读写操作完要unpin。**

一个==b+树节点包含头部（内部节点24B，叶子节点28B(多了个nextpageid)）和柔性数组（存储kv对）==
![](Pasted%20image%2020230520174055.png)
## 内部节点

![](Pasted%20image%2020230520174238.png)
![](Pasted%20image%2020230520174230.png)
![内部接单](Pasted%20image%2020230514134852.png)
![](Pasted%20image%2020230514134552.png)
数据存储方面，每个节点的理论结构应该是<指针，键，指针，键…，键，指针>。bustub使用一个存储键值对的柔性数组（等价于一个指针）进行存储`MappingType array[1]`。==为了实现该效果，将内部节点第一个key置为invalid==。

## 叶子节点

![](Pasted%20image%2020230514134926.png)
同样使用键值对柔性数组存储数据。但是第一个key有效。ValueType只有RID(record_id)，因此为非聚簇索引。即实际数据存放的位置。

# CheckPoint1

## Task1

实现三个类：
- B+Tree Parent Page：后两个类的基类
- B+Tree Internal Page：不存储真实数据。存储m个有序key和m+1个孩子指针。第一个key无效，查找从第二个key开始。至少半满
- B+Tree Leaf Page：叶节点的value是RID类型。在哪个磁盘Page中，在这个Page的哪个槽里

### 所需文件

src/include/storage/page/b_plus_tree_page.h
src/storage/page/b_plus_tree_page.cpp

src/include/storage/page/b_plus_tree_internal_page.h
src/storage/page/b_plus_tree_internal_page.cpp

src/include/common/rid.h  // 叶子value是RID类型的
src/include/storage/page/b_plus_tree_leaf_page.h
src/storage/page/b_plus_tree_leaf_page.cpp

## Task2

b+树只支持唯一key。
在更改根节点后，更新`root_page_id`

### 插入

涉及分裂
![](Pasted%20image%2020230515163401.png)

两种分裂方法：
	1. 开辟新中间数组，然后前部分给旧节点，后半部分给新节点
	2. 直接将旧节点后半部分给新节点。

### Delete

删除流程：(最困难，涉及合并和重分配)
1. 找到包含（K,P)的子节点N并进行删除。
2. 如果N是根节点并且删除后只剩下一个节点，那么将此节点变成新的根节点并删除N。
3. 如果N又太少的K或者P

假如存在一侧节点有富余的 KV 对，则成功偷取，结束操作。若两侧都没有富余的 KV 对，则选择一侧节点与其合并。



# CheckPoint2

这个主要是实现并发。3是访问，4是并发。

## Task3

实现支持for循环迭代的迭代器

### 文件

src/include/storage/index/index_iterator.h
src/index/storage/index_iterator.cpp

## Task4 latch crabing

使B+树索引支持并发访问
需要粒度更小的锁，分读写。

### 加锁步骤

-   先锁住 parent page，
-   再锁住 child page，
-   假设 child page 是_安全_的，则释放 parent page 的锁。_安全_指当前 page 在当前操作下一定不会发生 split/steal/merge。同时，_安全_对不同操作的定义是不同的，Search 时，任何节点都安全；Insert 时，判断 max size；Delete 时，判断 min size

delete除了判断安全方式使用minsize，还有可能对sibling节点加锁

![](591f869b02694cf18df4a87738ed71ee.png)


# 测试

![](Pasted%20image%2020230517182903.png)
![](Pasted%20image%2020230517182915.png)

# 难点

## 几种索引

==我们实现的是非聚簇索引，因为叶子节点存的不是数据是RID。==

聚簇索引、非聚簇索引，主键索引、二级索引（非主键索引）的区别:
>在聚簇索引里，leaf page 的 value 为表中一条数据的某几个字段或所有字段，一定包含主键字段。而==非聚簇索引 leaf page 的 value 是 record id，即指向一条数据的指针==。 在使用聚簇索引时，主键索引的 leaf page 包含所有字段，二级索引的 leaf page 包含主键和索引字段。当使用主键查询时，查询到 leaf page 即可获得整条数据。当使用二级索引查询时，若查询字段包含在索引内，可以直接得到结果，但如果查询字段不包含在索引内，则需使用得到的主键字段在主键索引中再次查询，以得到所有的字段，进而得到需要查询的字段，这就是回表的过程。 在使用非聚簇索引时，无论是使用主键查询还是二级索引查询，最终得到的结果都是 record id，需要使用 record id 去查询真正对应的整条记录。 聚簇索引的优点是，整条记录直接存放在 leaf page，无需二次查询，且缓存命中率高，在使用主键查询时性能比较好。缺点则是二级索引可能需要回表，且由于整条数据存放在 leaf page，更新索引的代价很高，页分裂、合并等情况开销比较大。 非聚簇索引的优点是，由于 leaf page 仅存放 record id，更新的代价较低，二级索引的性能和主键索引几乎相同。缺点是查询时均需使用 record id 进行二次查询。

在`**B_PLUS_TREE_INTERNAL_PAGE**`中使用`flexible array`柔性数组
![](Pasted%20image%2020230514012625.png)
为什么这么用？
内部节点和叶节点对象都不是直接创建出来的，而是由Buffer Pool管理的Page的data_(4KB char数组)使用reinterpret_cast转化过来的。
![](Pasted%20image%2020230514140034.png)


clang-tiey要求成员变量以_结尾。

## 关于节点size和分裂合并问题

内部节点的size将第一个无效的key计算在内。

插入kv对时，叶子节点和内部节点分裂的条件是不一样的。
	You should correctly perform splits if insertion triggers the splitting condition (number of key/value pairs AFTER insertion equals to max_size for leaf nodes, number of children BEFORE insertion equals to max_size for internal nodes).
![](Pasted%20image%2020230517203720.png)
看这副图，插入4之后，叶节点大小只有2 = maxsize，但是依旧发生了分裂。因为插入后size=maxsize并且是叶子节点所以发生了分裂(根据官网的定义来的)

叶子节点插入时，可以直接进行插入，不考虑是否满的情况。也为bustub设计中，叶子节点留了一个空隙。
内部节点没有留空隙不可以直接插入。因此先开一个比节点size大1的char数组来存储过满内部节点。
这是因为内部节点第一个key无效，直接插入有可能溢出。

假如我们有一棵 5 阶的 B+ 树。5 阶只是一种常用的说法，代表 B+ 树节点最多能容纳五个 KV 对。对于 leaf page 来说，当 B+ 树处于稳定状态时（插入、删除等操作已经完全结束），最多只能有 4 个 KV 对。对于 internal page，最多有 4 个 key，5 个 value，可以看成是有 5 个 KV 对。

叶子和内部节点最大最少kv对数量：
- 书中：
	- 内部节点指针数最少为N/2上取整，叶子节点(N-1)/2上取整

此b+树中内部节点和叶子节点maxsize不相同。

在分裂的时候要考虑溢出情况：
![](Pasted%20image%2020230518165808.png)
要注意叶节点maxsize分裂和内部节点maxsize+1分裂
这里容易堆溢出
![](Pasted%20image%2020230518170723.png)

这里容易

叶子的size算第一个吗？size == max + 1时分裂，那这意思就是算第一个。
## 加锁问题
### 为什么要对整颗树加锁

![](Pasted%20image%2020230520215105.png)
在确定根节点不会更新时可以对整棵树索引进行解锁


### 死锁问题

>如何记录我们对那些page加过锁？
>
>**使用traction参数如果一个 page 出现在 transaction 的 page set 中，就代表这个线程已经持有了这个 page 的锁。**

在持有多个锁时，都是从上往下获取锁，获取锁的方向是相同的，在获取sibling锁时，一定有其parent的锁。因此不会出现死锁。
**但是在delete时，会获取sibling的锁。在index iterator中，从左往右获取叶子锁，在merge和重分配时也需要获取左sibling锁，方向想反，可能死锁**

#### 解决办法

在 Index Iterator 无法获取锁时，应放弃获取。

### 修改子页的父指针要加锁吗

不需要，加了会死锁

# 亮点

要求使用粒度更小的锁和读写锁（使用std::mutex，性能远高于mutex）。不使用RAII锁使粒度更小。

> 使用柔性数组来存储b+树节点数据

必须是类最后一个（数组）成员，可以大小未知。
是连续的内存，可以提高访问速度。


`reinterpret_cast` 用于无关类型的强制转换，转换方法很简单，原始 bits 不变，只是对这些 bits 用新类型进行了重新的解读。可想而知这种转换非常不安全，需要确保转换后的内存布局仍是合法的。在这里原类型是 byte 数组，新类型是我们需要使用的 tree page。

# 优化

采用乐观锁。
在insert/delete时，对沿途节点上读锁并及时释放，left上写锁。当发现操作对 leaf page 确实不会造成 split/steal/merge 时，可以直接完成操作。当发现操作会使 leaf page split/steal/merge 时，则放弃所有持有的锁，从 root page 开始重新悲观地进行这次操作，即沿途上写锁。

不再对整棵树加锁，只锁被修改的分支

查找时采用二分查找/lowbound

