使用unique_ptr->get()获取其中保存的指针。

代码在bustub上。先git clone下来，然后切换到2022年左右的一个commit版本上。
将其保存成为2022分支。在2022_p0分支上完成本lab。
使用的时C++17。

# 目标

实现一个**KV存储的并发字典树**。
key是一个非空变长字符串，value是任意类型。
value存储在代表key的最后一个字母的节点中。

# 所需文件

`src/include/primer/p0_trie.h`
`test/primer/starter_trie_test.cpp`

# 类

三个类：TrieNode、TrieNodeWithValue、Trie。
第一个类是字典树上的一个任意类型节点、TrieNodeWithValue是字典树上的终端节点。
Trie是一颗字典树。

## TrieNode

```cpp
 protected:
  char key_char_;
  bool is_end_{false};
  std::unordered_map<char, std::unique_ptr<TrieNode>> children_;
```

## TrieNodeWithValue

```cpp
 private:
  T value_;  // 终端字典树节点相对普通节点多了value_是模板类型
```

## Trie

```cpp
class Trie {
 private:
  std::unique_ptr<TrieNode> root_;  // trie的根，独享指针
  ReaderWriterLatch latch_;  // bustub内部封装实现的一个读写锁
 public:
  template <typename T>
  auto Insert(const std::string &key, T value) -> bool;
  auto Remove(const std::string &key) -> bool ;
  template <typename T>
  auto GetValue(const std::string &key, bool *success) -> T ;
};
```

# 构建并本地测试

```
# 构建
mkdir build
cd build
cmake -DCMAKE_BUILD_TYPE=Debug ..
make -j8
// 本地测试
cd build
make starter_trie_test
./test/starter_trie_test
```

# 进行格式规范检查

## 命令

**Formattng**

```
make format  // clang-format
make check-lint  // 找出程序中的错误，这些错误包括潜在的语法错误，编译错误，拼写错误等
make check-clang-tidy-p0 // clang-tidy是一个基于clang的C++静态分析工具，主要用来检测代码中的常见错误。通过对代码运行静态分析，可以找到潜在的Bug或者代码风格的不一致问题。
```

**内存泄漏**

```
valgrind \
    --error-exitcode=1 \
    --leak-check=full \
    ./test/starter_trie_test
```

# 亮点和难点

## 难点

`auto InsertChildNode(char key_char, std::unique_ptr<TrieNode> &&child) -> std::unique_ptr<TrieNode> *`函数中使用std::unique_ptr作为形参