# 小技巧
在使用删除函数时，先检查是否为空。删除后如果是指针记得释放空间（delete）
c++整数相除自动向下取整，不是四舍五入。
在索引时，先查看索引是否合法。
char类型相减，得到的是ASCII差值。
vector int数组会自动初始化为0；
![](Pasted%20image%2020230318213118.png)
第一个是默认初始话，第二个是值初始化
全局静态变量默认初始化为0；

# 尚未分类零碎知识

decltype作用于某个函数时，返回函数类型而非指针类型。需要显式加`*`来表明需要返回指针。
当使用函数名字时，在需要的情况下会自动转化成一个指针，不需要取地址。
有序容器关键字必须支持比较操作。
vector迭代器支持比较操作。list不是连续存储，迭代器不支持比较操作。
返回值可以进行列表初始化。
对一个对端已经关闭的socket调用两次write, 第二次将会生成**SIGPIPE**信号,该信号默认结束进程。

## return 和 exit

- return是关键字，exit是标准库函数
- return结束当前函数的执行，并返回一个值给调用者；exit：终止进程并进行一些清理操作。
- main函数最终会调用return，然后结束main，将返回值传给exit，隐式调用exit

# 初始化

值初始化：内置类型初始化为0；类类型由类默认初始化。
值初始化定义时后面有对小括号，没有则进行默认初始化。

# 宏
INT_MN: -2147483647
INT_MAX: 2147483647
# string
是一种可变类型
## 常用函数

```c++
string::resize()
void string::erase(string::iterator iter) //删除迭代器所指的字符
void string::reverse(int n, char c) // 将字符串大小调整为n，不够的用字符c填充字符串也行。
string string::substr(int pos, int len)  //左闭右开，截取连续字串。
String operator+(const String &s)   //string中重载了+
string::size_type string::find(char c || string s) //返回找到的首位置，找不到返回string::npos
string::npos  //静态成员常量：是对类型为size_t的元素具有最大可能的值。当这个值在字符串成员函数中的长度或者子长度被使用时，该值表示“直到字符串结尾”。作为返回值他通常被用作表明没有匹配。string::npos是这样定义的：static const size_type npos = -1;
bool string::empty() 
char string::back()
char string::front()
`string.begin(), string.end();`
`void std::string::push_back()` 字符串最后添加一个字符。
`void std::string::pop_back()` 删除字符串最后一个字符。
`void std::sort(begin, end)`：对区间进行排序，参数是两个迭代器。
`string& replace (size_t pos, size_t len, const string& str);`：替换
```
## 与C中char []对比
在c语言中把一个字符串存入数组时，也把结束符`/0`存入了数组，并以此作为字符串结束的标志。
c++不再使用/0，使用size来判断。
``
## list和vector的区别
vector类似于数组
list是使用双向链表实现的，内存空间不连续，只能通过指针进行数据访问。

# 库/算法函数
![](Pasted%20image%2020230221191901.png)
```c++
swap(): 1.    
    template <class T> void swap ( T& a, T& b )
reverse() :     // 此函数区间为左闭右开。区间内反转迭代器所指对象。
    template <class BidirectionalIterator> void reverse (BidirectionalIterator first,            BidirectionalIterator last)
sort():   
    template <class RandomAccessIterator, class Compare>
    void sort (RandomAccessIterator first, RandomAccessIterator last, Compare comp);
    // 第三个参数是比较函数，可以是自己定义的。
stol stoll: 将string变成long int 或者long long int类型数据
int atoi(const char *str)
string to_string(int/short/long/... val) //将val变成string
ForwardIterator lower_bound (ForwardIterator first, ForwardIterator last,const T& val);
//     在 [first, last) 区域内查找不小于 val 的元素 ，找不到返回last找到返回该数的迭代器。
```

# 关联容器
==关键字是const的==

## 关联容器迭代器
解引用后得到的是一个value_type的类型
set的迭代器是const的，因为set关键字不能修改。
### 遍历关联容器
`while (t.begin() != t.end())`

## 函数
```c++
t.begin()
t.end()
key_type
mapped_type
value_type
find()
int erase(key_type) //删除所有给定关键字的元素，返回实际删除的元素数量。
```
## set
### 函数
set.find(T item):返回值是set< T  >::iterator。如果找到返回指向该值的迭代器指针，否则返回set.end()。
```c++
int x;
int* pos;
s.insert(x);//插入元素x
clear();//清除所有元素
s.erase(x);//删除元素x
erase(pos);//删除pos迭代器所指的元素，返回下一个元素的迭代器
s.size(x);//返回s的大小
s.count(x);//返回x元素出现次数     在set中无重复元素，因此count函数在set中类似于find
s.begin();//返回第一个元素的迭代器
s.end();//返回最后一个元素的迭代器的下一位
s.find(x);//查找x，若存在，返回该键的元素的迭代器；若不存在，返回s.end();
s.empty();//返回set是否为空
```
set是有序的，放入其中的元素默认按照key值升序进行排列。

## map
map是有序的吗？默认怎么排序？
如何按照value进行一个排序？

map元素的key不可以重复，key默认是有序的。map的元素是pair类型。
```c++
map[key] = value；
map.insert(pair<T, T>(item1, item2))；   // map插入元素的两种方式。
```

### pair类型

pair类型默认进行值初始化。数据成员分别为first和second，都是public的，使用成员访问符号（.）来进行访问。
#### 函数
```c++
p.first
p.second
make_pair(v1, v2)
```
# vector
## 创建及初始化
可以使用相同类型的set的两个迭代器来初始化。
## vector< char >
类似于string，但并未重载+运算符。
## 函数
```c++
vector.clear() //把size设置成0，capacity不变。
```
# 数组
数组的大小不可改变
如果要添加元素时，从后往前移动。

# 栈和队列(容器适配器)

![](Pasted%20image%2020230223223641.png)

## STL版本
 - HP STL
 - SGI STL
 - 最后一个不开源
==**我们讲解的是SGI STL**==

## 底层实现

stack /queue容器适配器的模板有两个参数。第一个参数是存储对象的类型，第二个参数是底层容器的类型。如果不写默认采用deque。

**栈是以底层容器完成其所有的工作，对外提供统一的接口，底层容器是可插拔的（也就是说我们可以控制使用哪种容器来实现栈的功能）。**
==STL中栈不是容器，而是一种容器适配器==

>
![](Pasted%20image%2020230223224334.png)
默认使用deque
>> 
>> 指定栈的底层实现容器
```c++
std::stack<int, std::vector<int>> third;  // 使用vector为底层容器的栈
```
队列默认也使用deque实现

==栈和队列都不允许有遍历行为，不提供迭代器==

> 指定list为队列的底层实现容器
```c++
std::queue<int, std::list<int>> third; // 定义以list为底层容器的队列
```

## 函数
### 栈函数

```c++
bool stack::empty()
T stack::top()
void stack::pop()
void stack::push()
int stack::size()
```
### 队列函数
```c++
queue<T> qu
bool qu.empty()
T qu.front()
T qu.back()
void qu.pop()
void qu.push()
int qu.size()
```


## 优先级队列
![](Pasted%20image%2020230301174624.png)
![](Pasted%20image%2020230301174638.png)
![](Pasted%20image%2020230301174645.png)
![](Pasted%20image%2020230301174655.png)
![](Pasted%20image%2020230301174708.png)
![](Pasted%20image%2020230301174716.png)


## 经典问题
### 栈
- 栈和队列的相互实现
- 括号匹配：**简单思路**实现
- linux cd命令进入多重目录
- 栈与递归的转换
- 表达式转换
### 队列
- 单调队列
- 优先级队列：底层是大小顶堆（完全二叉树）

# 树

## 大小顶堆
适用于找数据集中的前k个大/小元素。
是个完全二叉树
是优先级队列的底层实现数据结构。

## 二叉树

### 几种二叉树
#### 满二叉树
#### 完全二叉树
#### 二叉搜索树平衡二叉搜索树 
左<根<右。
AVL可以为空，左右两颗子树高度差不超过1.
c++中set、map、multiset、multimap底层实现都是平衡二叉搜索树（红黑树），，所以map、set的增删操作时间时间复杂度是logn，。unordered_map、unordered_set，unordered_map、unordered_set底层实现是哈希表。

### 二叉树存储方式
- 链式存储
- 顺序存储

#### 链式
![](Pasted%20image%2020230302145555.png)
#### 顺序存储
![](Pasted%20image%2020230302145627.png)
如果父节点数组下标是i，左子树数组下标是`2*i+1`，右子树下标是`2*i+2`；

### 二叉树遍历方式
1. 深度优先遍历（递归/迭代）
	- 前序：中左右
	- 中序：左中右
	- 后序：左右中
2. 广度优先遍历（借用队列迭代）
	- 层次遍历
#### 深度优先遍历
==这里的前中后序指的是根节点的访问顺序。==

迭代方式实现：
	需要借助栈

统一迭代方式 ：
	栈里面可以存储NULL，将访问过但是没有进行处理的节点下一个标记为NULL，当遇到NULL就说明需要取出结果了；
	将栈里面的存储顺序调整成需要的顺序然后按顺序出栈并加入结果集。
	当栈空时，不再进行迭代。










### 二叉树定义
```c++
struct TreeNode{
	int val;
	TreeNode* left;
	TreeNode* right;
	TreeNode(int x) : val(x), left(NULL), right(NULL) {}
};
```

