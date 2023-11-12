用来高效的**存储和检索字符串**集合的**多叉树**结构。
有两种操作：插入字符串和查找字符串。

逻辑结构示例：
![](Pasted%20image%2020231006225250.png)

代码实现：

>==数据结构==

`int son[N][26], cnt[N], idx;`， idx == 0，既是根节点也是空节点。

> 模板

```cpp
int son[N][26], cnt[N], idx;******

int insert(char str[]) {
	int p = 0;
	for (int i = 0; str[i]; i ++) {
		int u = str[i] - 'a';
		if (!son[p][u]) son[p][u] = ++ idx;
		p = son[p][u];
	}
	cnt[p] ++;
}

int query(char str[]) {
	int p = 0;
	for (int i = 0; str[i]; i ++) {
		int u = str[i] - 'a';
		if (!son[p][u]) return 0;
		p = son[p][u];
	}
	return cnt[p];
}
```