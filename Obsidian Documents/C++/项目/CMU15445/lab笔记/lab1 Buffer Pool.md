# 总览

整个实验要做一个==面向磁盘==的BusTub DBMS的存储管理器。
Buffer Pool的作用是在主存和磁盘之间来回搬运物理页。
系统通过page_id_t来请求页面。
要线程安全。

# TASK1 可扩展哈希表

## 简介

这是一种动态hash表，大小不固定。
可扩展hash有两种结构：
	- directory：用一个数组来存储桶指针`std::vector<std::shared_ptr<Bucket>> dir_`。目录大小（也即桶的个数）为2^global_depth_。类似于目录id，是二进制的（根据hash后的后global_depth_位来找桶号）。当桶进行分裂后，目录有可能会expansion。
	- bucket：用于存储hash表的数据。hash表是由多个bucket组成的，目录只是存储bucket的指针。local depth <= global depth。==桶的大小是固定的==，如果超过指定个数，bucket会进行分裂。
	- local depth：初始为0.局部深度是该桶里的记录用到几位哈希值。
	- global depth：初始为0.只有一个桶，直接往桶中放即可。代表所有记录最多用到几位hash值。
有的桶里的元素用不到全局深度那么多位hash值。

## 目标

实现一个可扩展哈希表。
不指定表的最大size，根据需求增长，但是不需要减小。

## 所需文件

`src/include/container/hash/extendible_hash_table.h`
`src/container/hash/extendible_hash_table.cpp`

## 实现

**可扩展哈希表类**
![](Pasted%20image%2020230511205037.png)
每个桶是一个share_ptr。分裂后生成两个新桶，dir进行重新指向hash。旧桶无dir元素指向后因为是share_ptr所以会自动析构。
**hash桶类**


包含两个类ExtendibleHashTable和Bucket。Bucket类是在ExtendibleHashTable类内进行定义的。

先进行hash然后与mask相与获得后global depth位作为桶索引
std::hash之后几乎没有会相等的。

使用scoped_lock，一种出作用域会自动析构的锁，对mutex进行了封装，c++14出现的区域锁。
![](Pasted%20image%2020230326230047.png)
为什么要这么做？GetGlobalDepthInternal这几个internal函数是private的。

## 可扩展哈希表的原理：

是一种动态hash。是一种数据库索引技术。

**与静态hash比较：**

与静态散列相比，动态散列的主要优势在于其性能不会随着记录数增长而下降，另外还具有最小的空间占用。缺点在于它会额外增加一次查询定位，因为在查询bucket本身前，需要先查找目录来定位bucket。  
另一种动态散列技术-线性散列(linear hashing)可以避免额外的查询定位，但可能这种方式需要更多的溢出桶，

### 名词解释

目录：是一个数组（vector），其中存放桶的指针，目录长度为`pow(2, global_depth)`。使用`std::hash()(key)`来获取桶号.
桶：用一个双向链表来实现的，存储的是真实的数。
![](Pasted%20image%2020230402205724.png)
global depth：目录的深度。即取hash后的key的最后几位作为桶的id号。
local depth：桶的深度，每个桶都有一个local depth。==如果局部深度小于全局深度时，一个桶可能包含不止一个指针指向它。==
桶分裂：每个桶的大小是给定的，当超过大小时会进行桶分裂，变成两个桶。在原来的基础上加入一个桶，将原来桶中的元素rehash的这两个桶中。
目录扩容后的指针指向：原目录指向不变。目录每次扩容增加一倍元素。新增加的按顺序进行指向。如原来是01，扩容后001和101（原来的后两位一样）指向同一个桶。也就是说后global depth一样的指向一个桶。

## 具体执行步骤

初始状态：全部深度局部深度都为0，数组中有一个元素，有1个桶。都是2^0 = 1；
插入元素：
1. 先对key进行hash，获得hash后的二进制值。
2. 截取此二进制的后global depth位，作为directory id。获取directory中的桶指针。
3. 开始向桶中插入元素。先看桶是否满了。如果桶已满，检查global depth和local depth。
		1. 如果local depth == global depth。自增global depth。目录倍增，将新增的目录项指向桶：dir_[i + capacity] = dir_[i];。新创建两个桶，其local depth为原来的+1，计算新的mask取hash值的倒数第global depth位，将原桶中的元素rehash到新桶中。遍历目录，如果目录项指向原来的桶，那么根据与mask的相与结果rehash，分别指向两个新桶。原桶无指针指向自动析构。
		2. local depth < global depth。新建两个local depth  + 1的桶，根据hash与1 << local depth相与，将原来同种元素rehash到新两个桶中。遍历目录，将指向原来桶的目录项，根据dir_id与1 << local depth相与的结果rehash，分别指向新的两个桶。原桶自动析构。
4. 进入这一步桶此时肯定是没有满的状态。有则更新，无则插入。
查找：
	根据当前的全局位深度，通过目录直接定位到桶地址，随后在桶内部逐一查找。
删除：
	对于删除操作，与查找操作类似，删除元素后，如果发现桶变为空，可与其兄弟桶进行合并，并使局部位深度减一。如果所有的局部位深度都小于全局位深度，则目录数组也进行收缩。

## 类

两个类：`class ExtendibleHashTable : public HashTable<K, V>`、`Bucket`。
bucket是ExtendibleHashTable的嵌套类

```cpp
class ExtendibleHashTable : public HashTable<K, V> {
 public:
 private:
  int global_depth_;    // The global depth of the directory
  size_t bucket_size_;  // The size of a bucket
  int num_buckets_;     // The number of buckets in the hash table 和目录大小一样
  mutable std::mutex latch_;
  std::vector<std::shared_ptr<Bucket>> dir_;  // The directory of the hash table
```

```cpp
class Bucket {
   private:
    size_t size_;                      // 桶的最大size
    int depth_;                        // 该桶的局部深度local_depth
    std::list<std::pair<K, V>> list_;  // bucket使用list实现，里面存储kv对。
};
```

## 基本操作

### 查

即`ExtendibleHashTable<K, V>::Find(const K &key, V &value)`函数实现：
1. 上锁
2. IndexOf(key)获取所在桶的目录id
3. 根据目录id在目录中找到桶的指针
4. target_bucket->Find(key, value)
5. `std::any_of(list_.begin(), list_.end(), [&key, &value](const auto &item))`

### 删

即`auto ExtendibleHashTable<K, V>::Remove(const K &key) -> bool`函数实现：

1. 上锁
2. 获取所在目录号，根据目录号获取桶指针
3. target_bucket->Remove(key)
4. std::any_of

### 增

`auto ExtendibleHashTable<K, V>::Bucket::Insert(const K &key, const V &value) -> bool`：

# TASK2 LRU-K

## 目标说明

用于跟踪缓存池中的页面使用情况。
所需实现的类和其他类没有关系。
实现LRU-K策略。

## 所需文件

1. src/include/buffer/lru_k_replacer.h -> LRUReplacer类
2. src/buffer/lru_k_replacer.cpp

## 理解

[讲解好的一个](https://blog.csdn.net/AntiO2/article/details/128439155)
### 什么是k距离

LRU-K。删除向后k距离最远的一个页面。
![](Pasted%20image%2020230509020033.png)
缓存池中出现不够k次，那么他的k距离为：+inf。

## 实现

lru_k置换器类
```cpp
class LRUKReplacer {
 private:
  size_t curr_size_{0};   // 缓存池中可以驱逐的页面数量（history + cached）
  size_t replacer_size_;  // frame_id为 【1， replacer_size_】
  size_t k_;              // lru_k算法，k参数
  
  std::mutex latch_;
  std::unordered_map<frame_id_t, size_t> access_count_;  // 某个页面的访问量
  std::list<frame_id_t> history_list_;  // 缓存池中但是没有到达k_次访问的页面。
  std::unordered_map<frame_id_t, std::list<frame_id_t>::iterator> history_map_;
  std::list<frame_id_t> cache_list_;  // 到达k次访问的缓存池中的页面；
  std::unordered_map<frame_id_t, std::list<frame_id_t>::iterator> cache_map_;  // 用于根据页面号快速找到某个页面的迭代器(类似指针)。新加入的页面放前面，先删除后面的。
  std::unordered_map<frame_id_t, bool> is_evictable_;  // 记录某个页面号是否可以驱逐。
};
```
### 驱逐策略

优先驱逐距离为+inf的frame，如果有多个，可以使用LRU、FIFO，本题使用FIFO。
> When multipe frames have +inf backward k-distance, the replacer evicts the frame with the earliest timestamp.

### 为什么使用LRU-K不用LRU

maybe_unused  c++17用于描述暂时没有被使用的函数和变量，以避免编译器发出警告。

## 难点

`RecordAccess`时达到k_ 次，放入缓存队列 ，**记住若小于 k_ 次，访问不会改变位置（FIFO驱逐his_list?）**，按第一次来，因为这个所以才通不过在线测试，且缓存的evictable_应该初始化为false, 否则影响测评。

# TASK3 缓存池管理器实例

## 任务

**概念区分**：
- ==**磁盘上的叫页page（也叫磁盘块），内存中的叫帧frame**==
- 页的大小是固定的，帧的大小和操作系统页相同。
- 页由一组连续的物理块组成，存储着数据库中的数据和元数据信息。在MySQL中，一个页包含多个数据行，每个数据行存储一条记录的数据，而帧存储从磁盘读取的页的副本；
- 页是数据库存储管理的基本单位，而帧是缓冲池中的基本单位；

实现类`BufferPoolManagerInstance`类。该类将会使用到task1、2中实现的数据结构。**可扩展hash table实现page_id到frame_id的映射**。**使用LRUKReplacer来跟踪访问Page对象的时间，以便在必须释放一个帧以腾出空间从磁盘复制新的物理页时决定要驱逐哪个对象**。
从磁盘管理器中取数据库页并存储到内存中。
刷脏页
读写磁盘代码在`DiskManager`类中已经实现。
Page对象包含内存块。Page对象的标识符(page_id)跟踪它所包含的物理页面，如果Page对象不包含物理页，那么它的page_id必须设置为INVALID_PAGE_ID。
每个Page对象还维护一个计数器，用于“pinned”该页的线程数。pin的页不可以free。每个Page还要记录它是不是脏页，如果是脏页，必须进行刷脏页写回磁盘。

## 优秀参考博客

https://blog.csdn.net/MelroseLbt/article/details/130175758?spm=1001.2014.3001.5502

## 理解

![](Pasted%20image%2020230512191330.png)
bustub 中有 Page 和 Frame 的概念。
Page 是承载 4K 大小数据的类，可以通过 DiskManager 从磁盘文件中读写，带有 page_id 编号，is_dirty 标识等信息。
Frame 不是一个具体的类，而可以理解为 Buffer Pool Manager（以下简称 BPM）中容纳 Page 的槽位，具体来说，BPM 中有一个 Page 数组，frame_id 就是某个 Page 在该数组中的下标。
![](Pasted%20image%2020230512005414.png)
![](Pasted%20image%2020230512005927.png)

## 文件

src/include/buffer/buffer_pool_manager_instance.h
src/buffer/buffer_pool_manager_instance.cpp
src/include/storage/page/page.h    // Page类定义  是主存中数据库的存储单元。

# 测试

步骤
```linux
修改test文件夹下的函数
删除DISABLE_

mkdir build
cd build
cmake -DCMAKE_BUILD_TYPE=Debug .. // 配置cmake DEBUG mode
make -j     // j后面跟需要的线程数。多线程加速编译

make lru_k_replacer_test -j8
./test/lru_k_replacer_test
```

### 结果

![](Pasted%20image%2020230511215236.png)

![](Pasted%20image%2020230509214448.png)


# 进一步优化

并发访问都是使用的一把大锁，尝试进行读写锁分离和减小所得粒度。

# 难点

可扩展哈希表中，目录倍增之后，新增加的目录指针指向哪个桶？
    新增的完全复制之前的，新增的和之前的不同是：之前的dir索引最高位是0，新增的最高位是1.
分裂后的两个新桶，原桶元素rehash是的local mask计算，怎么rehash原桶元素，怎么重新分配目录指针？
  1 << local depth(之前的)。因为原来桶中的数据后local depth位都是相同的，那么再只根据高一位进行rehash即可，其余都不关心
  ![](Pasted%20image%2020230512193154.png)
  将指向原桶的dir指针，重新使用local mask rehash。
  这个过程是一个while过程，因为有可能桶分裂之后，还是不够用，需要继续分裂重复整个过程。

使用perf进行性能分析
![](Pasted%20image%2020230512202752.png)


>进一步理解

在BufferPool中，帧号为0到pool_size-1。一开始所有帧号都在freelist中。
page_数组存放缓冲池中的帧中此时存放的磁盘页面，使用帧号来进行访问，即：**page_数组元素是磁盘页面，下标是该磁盘页面在缓冲池中存放的帧号。**
在NewPgImp中新分配一个帧给磁盘页面，这个磁盘页面号不能指定，是AllocatePage函数给的。页表中存放缓冲池中页号到帧号的映射，用于快速访问，比如，在取指定页面的时候，先去进行查找页表，根据映射关系找到所在帧号（即page数组下标号），然后在缓存池中根据帧号在page数组中取出该页面。