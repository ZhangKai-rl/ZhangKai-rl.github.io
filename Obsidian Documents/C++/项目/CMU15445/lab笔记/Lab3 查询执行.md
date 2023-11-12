# 背景

## SQL语句的执行

- parser使用DuckDB打包的libpg_query。通过libpg_query产生Postgres ==AST==后，
- Binder ：==将AST词语绑定到数据库实体上，生成Bustub AST。==将会把这个 AST 改写成 BusTub 可以理解的一个更高级的 AST，变成Bustub可以理解的AST。在这个过程中，我们会将所有 identifier 解析成没有歧义的实体。binder必须使用完整列名。
- Planner： ==递归遍历 Binder 产生的 BusTub AST，产生一个初步的树形查询计划。==planner中的表达式只能使用列的位置，如`#0.1`为第0个孩子的第一列。查询计划规定了数据的流向。数据从树叶流向树根，自底向上地流动，在根节点输出结果。
- Optimizer：有两种：
	1. Rule-based. 启发式。Optimizer 遍历初步查询计划，根据已经定义好的一系列规则，对 PlanNode 进行一系列的修改、聚合等操作。例如我们在 Task 3 中将要实现的，将 Limit + Sort 合并为 TopN。这种 Optimizer 不需要知道数据的具体内容，仅是根据预先定义好的规则修改 Plan Node。Bustub使用这种。
	2.  Cost-based. 这种 Optimizer 首先需要读取数据，利用统计学模型来预测不同形式但结果等价的查询计划的 cost。最终选出 cost 最小的查询计划作为最终的查询计划。
- Executor：遍历查询计划树，将树上的 PlanNode 替换成对应的 Executor。算子的执行模型也大致分为三种：
	1.  Iterator Model，或 Pipeline Model，或火山模型。每个算子都有 `Init()` 和 `Next()` 两个方法。`Init()` 对算子进行初始化工作。`Next()` 则是向下层算子请求下一条数据。当 `Next()` 返回 false 时，则代表下层算子已经没有剩余数据，迭代结束。可以看到，火山模型一次调用请求一条数据，占用内存较小，但函数调用开销大，特别是虚函数调用造成 cache miss 等问题。Bustub使用火山模型。
	2.  Materialization Model. 物化模型。所有算子立即计算出所有结果并返回。和 Iterator Model 相反。这种模型的弊端显而易见，当数据量较大时，内存占用很高。但减少了函数调用的开销。比较适合查询数据量较小的 OLTP workloads。
	3.  Vectorization Model. 对上面两种模型的中和，一次调用返回一批数据。利于 SIMD 加速。目前比较先进的 OLAP 数据库都采用这种模型。
算子的执行方向也两种：
	1.  Top-to-Bottom. 从根节点算子开始，不断地 pull 下层算子的数据。
	2.  Bottom-to-Top. 从叶子节点算子开始，向上层算子 push 自己的数据。Bustub使用。

==planner 生成 logical plan node，然后通过 optimizer 框架做很多步优化产生 physical plan node。==但是 BusTub 只是个教学项目，所以我们只有 physical plan node。Planner 会直接把 join plan 成 NestedLoopJoin，在 optimizer 里面改写成 HashJoin 或者 NestedIndexJoin
![](2307739-20210921111615561-153148691.jpg)
上层组件与Execution Engine交互。
在Execution Engine中，首先调用Execution Factory递归创建Executor。然后不断调用Executor的Next()方法，生成Tuple。
在创建Executor时，需要提供Execution Plan与Execution Context。前者可以看作是Executor Type Specific的创建参数（包含children、predicate、table_oid等），后者可以看作是Executor Type Irrelavent的创建参数（包含各种manager、catalog等）。
![](2307739-20210921112428008-1496644603.jpg)
Index是对lab2中B+ Tree Index的套壳。TableHeap本质上是由Page组成的链表。在每个page中，按slot存储tuple中的data。TableHeap类本身并不存储schema，schema存储在catalog中。Schema的本质是vector of columns。它描述了一个tuple内存储的数据。
Tuple存储了一行数据。它包含Data和RID。**注意，tuple中不存储schema**；TableHeap中不直接在page中保存tuple，而是保存serialize后的tuple data。
Value表示一个值（Int、Char、Varchar等）。用一个Value vector和一个schema，可以构建一个tuple。

Expression以树的形式递归地表示一个表达式。Expression共包含Const、Comparison、Column、Aggregation四种类型，分别用于不同的场合。下面加以详细阐述：
![](2307739-20210921114651582-1226723733.jpg)
Expression不支持AND/OR，也不支持四则运算。只支持最基本的Comparison类型。
Const Expression最简单，永远返回一个常数。
Column Expression接受一个tuple做参数，返回这个tuple中特定column处的值。
Aggregate Expression既可以表示一个GroupBy值(如Col2)，也可以表示一个Aggregation值（如Max(Col1)、Sum(Col2)等）。
Const、Column、Aggregation 三种Expression负责给Comparison Expression提供操作数。

### 简述

SQL 语句 parse 后，通过 binder 绑定 identifier 到各个实体，使用 planner 生成查询计划，而后使用 optimizer 优化成最终的查询计划。
## 遍历方式

Seq Scan：就是按照表的记录的排列顺序从头到尾依次检索扫描，每次扫描要取到所有的记录。这也是最简单最基础的扫表方式，扫描的代价比较大；
Index Scan：对于给定的查询，先扫描一遍索引，从索引中找到符合要求的记录的位置(指针)，再定位到表中具体的Page去取，即先走索引，再读表数据；
## 源码阅读

### ExecutorContext

```cpp
class ExecutorContext {
 private:
  /** The transaction context associated with this executor context */
  Transaction *transaction_;
  /** The datbase catalog associated with this executor context */
  Catalog *catalog_;
  /** The buffer pool manager associated with this executor context */
  BufferPoolManager *bpm_;
  /** The transaction manager associated with this executor context */
  TransactionManager *txn_mgr_;
  /** The lock manager associated with this executor context */
  LockManager *lock_mgr_;
};
```

### catalog

#### catalog

**bustub有一个全局单例的catalog类对象，管理bustub中所有的创建的index和table**

DBMS将bustub数据库的元数据存储在内部的catalog中。
该对象维护了整个数据库系统中创建过的==表、索引==等实例,并且提供了访问这些实例的接口,也就是说,如果在User代码中想要获取或设置某个Table或者Index对象,都需要先通过Catalog中的get或set函数来进行.

一个数据库维护了一个内部的 Catalog（目录）来记录数据库的元数据。比如，可以被用来回答当前有哪些表存在，它们的位置在哪，表中 Column 的类型，在2022 fall的实验中，并没有让我们实现一个 Catalog，但开始实验前仍然需要对此有了解。
```cpp
class Catalog{
  private:
  [[maybe_unused]] BufferPoolManager *bpm_;
  [[maybe_unused]] LockManager *lock_manager_;
  [[maybe_unused]] LogManager *log_manager_;
  /**
   * Map table identifier -> table metadata.
   *
   * NOTE: `tables_` owns all table metadata.
   */
  std::unordered_map<table_oid_t, std::unique_ptr<TableInfo>> tables_;
  /** Map table name -> table identifiers. */
  std::unordered_map<std::string, table_oid_t> table_names_;
  /** The next table identifier to be used. */
  std::atomic<table_oid_t> next_table_oid_{0};  给新创建的表的id
  /**
   * Map index identifier -> index metadata.
   *
   * NOTE: that `indexes_` owns all index metadata.
   */
  std::unordered_map<index_oid_t, std::unique_ptr<IndexInfo>> indexes_;
  /** Map table name -> index names -> index identifiers. */
  std::unordered_map<std::string, std::unordered_map<std::string, index_oid_t>> index_names_;
  /** The next index identifier to be used. */
  std::atomic<index_oid_t> next_index_oid_{0};
};
```

Catalog 提供了一系列 API，例如 `CreateTable()`、`GetTable()` 等等。Catalog 维护了几张 hashmap，保存了 table id 和 table name 到 table info 的映射关系。
table id 由 Catalog 在新建 table 时自动分配，table name 则由用户指定。系统的其他部分想要访问一张 table 时，先使用 name 或 id 从 Catalog 得到 table info，再访问 table info 中的 table heap。

**全局元数据不仅仅通过Catalog对象的方式维护在内存中,也需要有持久化在磁盘中的方式**.BusTub中定义了HeaderPage,也就是说会有一个HeaderPage文件持久化了这些元数据的内容.

##### indexinfo

```cpp
struct IndexInfo {
  /** The schema for the index key */
  Schema key_schema_;
  /** The name of the index */
  std::string name_;
  /** An owning pointer to the index */
  std::unique_ptr<Index> index_;
  /** The unique OID for the index */
  index_oid_t index_oid_;
  /** The name of the table on which the index is created */
  std::string table_name_;
  /** The size of the index key, in bytes */
  const size_t key_size_;
};
```

类似的IndexInfo 维护的是一个索引表的所有元数据（Metadata），比如该索引 key 所在表的名称，索引名称、索引唯一表示 id 等信息。

##### tableinfo

TableInfo 维护的是一张表的所有元数据（Metadata），比如一张表的 schema，表的名称，表的唯一标识 id，以及指向 table heap 的指针。
```cpp
struct TableInfo {
  /** The table schema */
  Schema schema_;
  /** The table name */
  const std::string name_;
  /** An owning pointer to the table heap */
  std::unique_ptr<TableHeap> table_;
  /** The table OID */
  const table_oid_t oid_;
};
```
table heap 是管理 table 数据的结构，包含 `InsertTuple()`、`MarkDelete()` 一系列 table 相关操作。table heap 本身并不直接存储 tuple 数据，tuple 数据都存放在 table page 中。table heap 可能由多个 table page 组成，仅保存其第一个 table page 的 page id。需要访问某个 table page 时，通过 page id 经由 buffer pool 访问。

==这张图非常重要，体现了一张表的存储方式==
![](Pasted%20image%2020230522010614.png)
#### schema

通常将每个表信息放在一个schema中，规定一张表是怎么存储数据的。

![](Pasted%20image%2020230522005859.png)
总的来说，schema 指明了一个 tuple 中的 value 部分存储的是什么数据。更具体点，数据库输出了一个数字100，我需要直到100是什么意思，是100元呢？还是100分？这部分是由 schema 规定的。

```cpp
class Schema {
 private:
  /** Fixed-length column size, i.e. the number of bytes used by one tuple. */
  uint32_t length_;   不太懂；可能是一个tuple的长度
  /** All the columns in the schema, inlined and uninlined. */
  std::vector<Column> columns_;  所有列
  /** True if all the columns are inlined, false otherwise. */
  bool tuple_is_inlined_{true};
  /** Indices of all uninlined columns. */
  std::vector<uint32_t> uninlined_columns_;  非内联列的索引
};
```
也就是一张表所有列的元信息。

#### Column

```cpp
class Column {
  /** Column name. */
  std::string column_name_;  列名
  /** Column value's type. */
  TypeId column_type_;  列值类型
  /** For a non-inlined column, this is the size of a pointer. Otherwise, the size of the fixed length column. */
  uint32_t fixed_length_;
  /** For an inlined column, 0. Otherwise, the length of the variable length column. */
  uint32_t variable_length_{0};  
  /** Column offset in the tuple. */
  uint32_t column_offset_{0};  在一个tuple中此列的起始偏移
};
```

### page

两个类：page和TablePage.
TablePage是Page的派生类，展现了一种table在文件中的常规的表示方式。

#### page

在BusTub中,一个Page对象对应磁盘上的一个block,其大小为4kb,只不过前者是加载于内存上的概念,后者是在磁盘上的概念.因此一个Page对象中,也会有一个buffer(4kb大小)来存储数据,也就是下面的data_。
```cpp
  char data_[BUSTUB_PAGE_SIZE]{};  // 对用磁盘上物理页（block）大小默认为4KB。内存中叫page，磁盘上叫block
  /** The ID of this page. */
  page_id_t page_id_ = INVALID_PAGE_ID;
  /** The pin count of this page. */
  int pin_count_ = 0;
  /** True if the page is dirty, i.e. it is different from its corresponding page on disk. */
  bool is_dirty_ = false;
  /** Page latch. */
  ReaderWriterLatch rwlatch_;
};
```

#### tablepage

page类的派生类。代表着存储引擎中的tablepage（一个物理页/块）。
**多个tablepage，使用链表形式组织（也可以使用页目录），形成一个堆。**
每个tablepage包含两部分：页头和页元组。
页头包含：页id，lsn，上页/下页id，空闲slot指针（也就是说tuple使用的槽页而不是日志结构），总tuple, tuple_offset  tuple_size

```cpp
/**
 * Slotted page format:
 *  ---------------------------------------------------------
 *  | HEADER | ... FREE SPACE ... | ... INSERTED TUPLES ... |
 *  ---------------------------------------------------------
 *                                ^
 *                                free space pointer
 *
 *  Header format (size in bytes):
 *  ----------------------------------------------------------------------------
 *  | PageId (4)| LSN (4)| PrevPageId (4)| NextPageId (4)| FreeSpacePointer(4) |
 *  ----------------------------------------------------------------------------
 *  ----------------------------------------------------------------
 *  | TupleCount (4) | Tuple_1 offset (4) | Tuple_1 size (4) | ... |
 *  ----------------------------------------------------------------
 *
 */
```
也就是说这个类，是实际存储tuple的地方。

> tuple的insert、updat、delete操作

- Insert:
    - 从0开始遍历slot,直到找到一个slot中的Tuple size为empty.
    - 根据该Tuple的size向低处移动FreeSpacePointer,并将Tuple中的数据拷贝.
    - 设置好该slot下的size和offset.
- Update,需要调整该slot对应的offset、size.并且其对应的数据也要重置,此外还需要将FreeSpacePointer还有高处的Tuple进行移动,并且重置offset.
- **Delete**.分为了两个阶段,一个是`MarkDelete`(只是将Tuple size中的最高位设置为1),另一个是`ApplyDelete`:清空对应的数据区、HEADER中的Tuple size和Tuple offset;移动FreeSpacePointer指针,并且调整其他Tuple的offset.
#### header_page

page_id = 0的page，用来存储元数据
### table

bustub中一个文件（tableheap）对应了一个表（但一个表可以由多个文件组成）

##### TableHeap

==**一个TableHeap代表了一张表**，可能由多个tablepage组成（就是Page）==。一个tableheap是一个存储在磁盘上的table的抽象。
也就是说页（物理块）的组织方式是：堆文件。

```cpp
class TableHeap {
publid :
  /** @return the begin iterator of this table */
  auto Begin(Transaction *txn) -> TableIterator;
  /** @return the end iterator of this table */
  auto End() -> TableIterator;
private:
  BufferPoolManager *buffer_pool_manager_;  TableHeap相关的API内部实现中,涉及到对Page的访问,其中获取某个TablePage的操作,往往是通过一个BufferPoolManager(缓存池)完成的
  LockManager *lock_manager_;
  LogManager *log_manager_;
  page_id_t first_page_id_{};   保存这张表起始page号，也就是说堆内页的组织方式是链表方式
};
```

##### TableIterator

```cpp
 private:
  TableHeap *table_heap_;
  Tuple *tuple_;
  Transaction *txn_;
};****
```

##### tuple

```cpp
 * Tuple format:  可以存储变长的或者定长的。varchar类型将地址和值的存储相分离
 * ---------------------------------------------------------------------
 * | FIXED-SIZE or VARIED-SIZED OFFSET | PAYLOAD OF VARIED-SIZED FIELD |
 * ---------------------------------------------------------------------
  bool allocated_{false};  // is allocated?
  RID rid_{};              // if pointing to the table heap, the rid is valid
  uint32_t size_{0};      tuple大小
  char *data_{nullptr};   实际存储这个tuple数据的地方
```
### type

包含两大类：value和type。
#### value

value表示具体的值，即一个tuple中一个字段的值。
存放具体的值，使用枚举类型typeid和联合体val，来表示此值的类型。因为这表示一个值，所以不需要基类派生类。
对应的是数据库字段（列值）。
使用ValueFactory类工厂模式，来生成具体的值。
#### type

表示bustub数据库中value的类型。
因为代表着类型，所以使用单例模式实现，存放在类型基类的静态数组中
### plan和expression

#### 关系

- `AbstractPlanNode`抽象各种不同的执行计划，包括 `SeqScanPlanNode`、`HashJoinPlanNode`、.....
- `AbstractExpression` 抽象了 sql 中的各种表达式，包括 `ArithmeticExpression`、`ColumnValueExpression`、`ComparisonExpression`、`ConstantValueExpression` 和 `LogicExpression`。

对于每一种 Executor 而言，都有一个 `AbstractPlanNode` 与之对应，而在每一个 `AbstractPlanNode` 都表明该计划的算法，通过 `AbstractExpression` 表明如何操作从儿子数据（最开始我们提到执行计划是一棵树）。每一个执行计划节点都会有 `AbstractExpression`，而这个表达式是以表达式树的形式展现的，执行节点的`AbstractExpression`是该表达式树的根节点。

![](Pasted%20image%2020230522011505.png)

#### expression

所有表达式类继承自`AbstractExpressionRef`虚基类。
最重要的函数是`Evaluate`
如column value表达式，其直接获取元组某一列的值。
多看上面的图。

### Optimizer

所有函数都定义在`optimize.h`文件中。使用`Optimize`函数调用其他优化函数对计划节点进行优化。

Optimizer的工作是让未经优化的原始 plan 树依次经历多条规则，来生成优化过的 plan。
```cpp
auto Optimizer::OptimizeCustom(const AbstractPlanNodeRef &plan) -> AbstractPlanNodeRef {
  auto p = plan;
  p = OptimizeMergeProjection(p);
  p = OptimizeMergeFilterNLJ(p);
  p = OptimizeNLJAsIndexJoin(p);
  p = OptimizeNLJAsHashJoin(p);  // Enable this rule after you have implemented hash join.
  p = OptimizeOrderByAsIndexScan(p);
  p = OptimizeSortLimitAsTopN(p);  // what we should add
  return p;
}
```

#### 小表驱动大表

小表驱动大表的主要目的是通过减少表连接创建的次数，加快查询速度。

优化 JoinOrder，用小表驱动大表。用`EstimatedCardinality()`函数估计表的大小，交换左右计划并重写predicate。另外注意重写后的outputSchema是不变的。在join的init用一个reordered_标志标记是否被reorder，具体做法就是通过outputSchema第一列的列名判断。在`next()`中构造返回值时判断如果reorder交换左右tuple顺序即可。

#### 谓词下推

观察 Filter 算子的下推路径，我们发现可以分解为两个步骤：
1.  将 Join 上面的 Filter 谓词下推到 Join 的连接条件上
2.  将 Join 的连接条件下推到左右子节点中
![](Pasted%20image%2020230522013215.png)
具体的，把所有Join的条件全收集起来，然后区分哪些是 Join 的等值条件，哪些是 Join 需要用到的条件，哪些全部来自于左子节点，哪些全部来自于右子节点。

区分之后，对于内连接，可以把左条件，和右条件，分别向左右子节点下推。等值条件和其它条件保留在当前的 Join 算子中，剩下的返回。

1.  ↙️ 谓词只包含来自左侧的列，可以下推到左子树中
2.  ↘️ 谓词只包含来自右侧的列，可以下推到右子树中
3.  ⏹️ 谓词同时包含左右两边的列，无法下推

主要就是dfs遍历表达式树，然后进行一些讨论，然后重写表达式树。

### RID

```CPP
class RID {
 private:
  page_id_t page_id_{INVALID_PAGE_ID};
  uint32_t slot_num_{0};  // logical offset from 0, 1...
};
```
唯一标识一个元组。

### PlanNode

计划节点都在plans文件夹中定义。
所有计划节点的虚基类父类是`AbstractPlanNode`.
```cpp
class AbstractPlanNode {
  /**
   * The schema for the output of this plan node. In the volcano model, every plan node will spit out tuples,
   * and this tells you what schema this plan node's tuples will have.
   */
  SchemaRef output_schema_;
  /** The children of this plan node. */
  std::vector<AbstractPlanNodeRef> children_;
};
```
## Bustub执行过程

![](Pasted%20image%2020230522001439.png)

![](v2-5266ee42751b14cb3353cb3334367e94_r.jpg)

# 与前两个lab的关系

例如 SeqScan 算子，需要遍历 table，首先通过数据库的 catalog 找到对应的 table，一个 table 由许多 page 组成，在访问 page 时，就用到了 Buffer Pool。在 Optimizer 中，假如发现 Sort 算子在对 indexed attribute 排序，会将 Sort 算子优化为 IndexScan 算子，这样就用到了 B+Tree Index。

# 两个关键函数

实现的算子中主要是init和next两个函数

## init

在初次调用算子时调用，进行初始化工作例如scan时将迭代器指向begin。

## next

每次调用时传出元组及其RID并返回true，否则返回false

# 任务概览

==实现查询执行器和优化器==

executor 本身并不保存查询计划的信息，应该通过 executor 的成员 plan 来得知该如何进行本次计算，例如 SeqScanExecutor 需要向 SeqScanPlanNode 询问自己该扫描哪张表。

==所有要用到的系统资源，例如 Catalog，Buffer Pool 等，都由 `ExecutorContext` 提供。==

创建执行SQL查询和实现优化器规则来转换查询计划
![“Heloise”](Pasted%20image%2020230519012331.png “12111”)
<center>Bustub数据库架构</center>
查询处理层已经提供了。
[示例DB Shell](https://15445.courses.cs.cmu.edu/fall2022/bustub/)只支持INT和VARCHAR类型。
EXPLAIN命令的结果提供了查询处理层内转换过程的概览。先解析器在绑定器，后者生成ast
查询计划：#0.0代表第一个元素的第一列。

使用火山模型。火山模型的优点在于占用内存较小，一次只使用一条数据，但这也导致我们会频繁调用函数，特别是虚函数，带来较大开销，而我们在执行查询时，也是自上向下请求数据。最终，当根节点获得了所有需要查询的数据时，就实现了 SQL 语句的功能。
RID是元组的唯一标识符

### 样例算子

Projection、Filter、Values。
阅读相关源码学习。

# TASK1: Access Method Executors

## 实现算子

SeqScan、Insert、Delete、IndexScan、update

==只有这几个执行引擎会和存储引擎进行交互==，会直接调用存储引擎接口的.这些操作所依赖的数据往往需要从disk(或者buffer)拉取,或者对这些数据需要写入到数据库中.而其他类型的Executor所依赖的数据往往不是数据库中的“实体数据“,是一些从子执行器中生成的,因此也不会直接访问存储引擎.

## ==表结构==

![](Pasted%20image%2020230521210754.png)

首先，Bustub 有一个 Catalog。Catalog 提供了一系列 API，例如 `CreateTable()`、`GetTable()` 等等。Catalog 维护了几张 hashmap，保存了 table id 和 table name 到 table info 的映射关系。table id 由 Catalog 在新建 table 时自动分配，table name 则由用户指定。

这里的 table info 包含了一张 table 的 metadata，有 schema、name、id 和指向 table heap 的指针。系统的其他部分想要访问一张 table 时，先使用 name 或 id 从 Catalog 得到 table info，再访问 table info 中的 table heap。

table heap 是管理 table 数据的结构，包含 `InsertTuple()`、`MarkDelete()` 一系列 table 相关操作。table heap 本身并不直接存储 tuple 数据，tuple 数据都存放在 table page 中。table heap 可能由多个 table page 组成，仅保存其第一个 table page 的 page id。需要访问某个 table page 时，通过 page id 经由 buffer pool 访问。

table page 是实际存储 table 数据的结构，父类是 page。相较于 page，table page 多了一些新的方法。table page 在 data 的开头存放了 next page id、prev page id 等信息，将多个 table page 连成一个双向链表，便于整张 table 的遍历操作。当需要新增 tuple 时，table heap 会找到当前属于自己的最后一张 table page，尝试插入，若最后一张 table page 已满，则新建一张 table page 插入 tuple。table page 低地址存放 header，tuple 从高地址也就是 table page 尾部开始插入。

tuple 对应数据表中的一行数据。每个 tuple 都由 RID 唯一标识。RID 由 page id + slot num 构成。tuple 由 value 组成，value 的个数和类型由 table info 中的 schema 指定。

value 则是某个字段具体的值，value 本身还保存了类型信息。

## seqscan

没有索引下的全表扫描使用
## insert

![](Pasted%20image%2020230523151348.png)
来源有两种：
- 1.VALUE算子：``INSERT INTO tbl_user VALUES (1, 15), (2, 16)``
- 2. 其余子执行器算子：`INSERT INTO tbl_user1 SELECT * FROM tbl_user2`
因此insert算子，必有一个子算子。使用while调用子算子next函数，返回所有元组后进行操作。

## delete

![](Pasted%20image%2020230523151406.png)

## indexscan

创建了索引的情况下进行全表扫描

### 算子实现步骤

1. 使用算子上下文和indexscan计划节点创建indexscan算子。进入init
2. 获取到该表的index_info
3.  根据index_info获取Index，并强转成BPlusTreeIndex
4. 遍历该索引树，并将所有rid加入数组中，将迭代器指向数组begin。进入next
5. 根据rid和事务信息，使用函数  `table_info->table_->GetTuple(*rid, tuple, txn);`获取该元组并传出到参数中。
6. 自增迭代器并返回。

# TASK2: Aggregation and Join Executors

Aggregation、NestedLoopJoin、NestedIndexJoin 三个算子。
## aggregation

==`enum class AggregationType { CountStarAggregate, CountAggregate, SumAggregate, MinAggregate, MaxAggregate };`==

处理聚集函数和GROUP BY.

以下查询会用到：
![](Pasted%20image%2020230523151535.png)
==本实验使用的是hash聚集，假设聚合hash表能放入内存（也不需要缓存池）。因此不需要多次hash==
原本是用了两个以上的hash函数，进行两次以上的hash。为了简化使用了一次hash，并且直接在hash表里存数据而不是桶。

### aggregatin_executor

#### 问题

##### pipeline breaker

==聚集算子是流式管道执行方式的破坏者。==
在init函数里（而不是next函数里）使用了一个while循坏获取子算子的所有next结果元组，并将其放到hash表中。
然后再在next函数里面一个一个emit返回元组。

##### NULLs

怎么处理空值？

##### HAVING

HAVING子句使用filter算子来完成。

```cpp
class AggregationExecutor : public AbstractExecutor {
 private:
  /** The aggregation plan node */
  const AggregationPlanNode *plan_;
  /** The child executor that produces tuples over which the aggregation is computed */
  std::unique_ptr<AbstractExecutor> child_;
  /** Simple aggregation hash table */
  SimpleAggregationHashTable aht_;
  /** Simple aggregation hash table iterator */
  SimpleAggregationHashTable::Iterator aht_iterator_;
};
```

#### SimpleAggregationHashTable

这个是聚集算子中使用hash算法的具体hash表结构。
即驻存在内存中的hash聚集表。

```cpp
class SimpleAggregationHashTable {
public :
  void InsertCombine(const AggregateKey &agg_key, const AggregateValue &agg_val);
 private:
  /** The hash table is just a map from aggregate keys to aggregate values */
  std::unordered_map<AggregateKey, AggregateValue> ht_{};
  /** The aggregate expressions that we have */
  const std::vector<AbstractExpressionRef> &agg_exprs_;
  /** The types of aggregations that we have */
  const std::vector<AggregationType> &agg_types_;
};
```
主要是这个kv map。key代表group by的字段的数组，value是需要aggregate字段的数组。
在下层算子传来一个 tuple 时，将 tuple 的 group by 字段和 aggregate 字段分别提取出来，调用 `InsertCombine()` 将 group by 和 aggregate 的映射关系存入 `SimpleAggregationHashTable`。若当前 hashmap 中没有 group by 的记录，则创建初值；若已有记录，则按 aggregate 规则逐一更新所有的 aggregate 字段。
![](Pasted%20image%2020230523220142.png)

### NestedLoopJoin

使用的join算法是NestedLoopJoin。**只处理左连接和内连接。**
**默认左表为外表。左表每和整个右表join完一项返回一次，所以不是pipeline breaker。**

**两种实现方式：**
- init阶段把join结果生成了并用vector保存着，next阶段遍历vector返回结果。类似于aggregation。这种要考虑数据能不能存储到内存中。这是一种pipeline breaker。
- next阶段再遍历左表和右表。
#### schema

Join 输出的 schema 为 left schema + right schema。
### NestedIndexJoin

子算子变成了一个，即外表。内表直接将外表项作为key使用index查找。可以看出，只有等值join，且有内表（右表）索引时才会优化成这个。

#### schema

即该算子输出的tuple的格式:输出的 schema 为 left schema + right schema。
	The schema of `NestedIndexJoin` is all columns from the left table (child, outer) and then from the right table (index, inner).

# TASK3: Sort + Limit Executors and Top-N Optimization

包含 Sort、Limit、TopN 三个算子，以及实现将 Sort + Limit 优化为 TopN 算子。

## 假设

假设每个ORDER BY的键都不重复。假设都能放入内存中。

## SORT

是个pipeline braker。在init中读取所有下层算子tuple，并按ORDER BY的字段排序。
需要在plan中拿出order_bys来，这是一个kv对的vector。代表<排序方式，排序表达式>

这个plan node不改变schema。输入输出schema一样。之前的是在plan node中已经规定好改成什么样了？
主要任务是重载比较运算符。

## LIMIT

这个plan node也不改变schema。
很简单。

## Top-N Optimization Rule

需要修改优化器。limit和sort一起出现时，优化成topn计划。

### 优化器

实现优化器

### 数据结构

使用什么数据结构找前几大/小的数据？
==使用优先级队列，大根堆。==
# Leaderboard Task

为 Optimizer 实现新的优化规则，包括 Hash Join、Join Reordering、Filter Push Down、Column Pruning 等等，让三条诡异的 sql 语句执行地越快越好。
## QUERY1

### join优化

#### hash join

==重要==
只能用于等值连接。
仅考虑in-memory情况。
build阶段在init，遍历左表（外表）建立hashmap，probe阶段在next进行，遍历右表（内表）探测是否有match的tuple。

hashmap的键应该为value。

#### join reorder

调用`EstimatedCardinality()`来估计table的大小。
尚未实现

#### index NLJ

## QUERY2

### filter pushdown





# 亮点

## 工厂模式

创建型模式。负责创建对象。

### AbstractExecutor

```cpp
auto ExecutorFactory::CreateExecutor(ExecutorContext *exec_ctx, const AbstractPlanNodeRef &plan)
    -> std::unique_ptr<AbstractExecutor> {
  switch (plan->GetType()) {
    // Create a new sequential scan executor
    case PlanType::SeqScan: {
      return std::make_unique<SeqScanExecutor>(exec_ctx, dynamic_cast<const SeqScanPlanNode *>(plan.get()));
    }
    // Create a new index scan executor
    case PlanType::IndexScan: {
      return std::make_unique<IndexScanExecutor>(exec_ctx, dynamic_cast<const IndexScanPlanNode *>(plan.get()));
    }
    // .....
}
```

## 解释器模式

行为型设计模式

```cpp
class AbstractExpression {
public:
    virtual int interpreter(map<char, int> Value)=0;
    virtual ~Expression(){}
};
//变量表达式
class ValueExpression: public AbstractExpression {
    char key;
public:
    ValueExpression(const char& key) {
        this->key = key;
    }
    int interpreter(map<char, int> Value) override {
        return Value[key];
    }
    
};
//符号表达式    两目运算符
class SymbolExpression : public AbstractExpression {
    // 运算符左右两个参数
protected:
    AbstractExpression* left;
    AbstractExpression* right;
public:
    SymbolExpression( AbstractExpression* left,  AbstractExpression* right):
        left(left),right(right){
    }
    
};
//加法运算
class AddExpression : public SymbolExpression {
public:
    AddExpression(Expression* left, AbstractExpression* right):
        SymbolExpression(left,right){
    }
    int interpreter(map<char, int> Value) override {
        return left->interpreter(Value) + right->interpreter(Value);
    }
};
```
