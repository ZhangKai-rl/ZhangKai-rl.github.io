# DFS

有两个重要概念：回溯、剪枝。
**每个DFS对应一个搜索树**。
算法思路比较奇怪的、空间复杂度要求高的

最重要的是暴力搜索的**顺序**。
回溯时比较重要的一点是恢复现场。
# BFS

适用于权重为1（相等）的最短路
# 经典问题
## 迷宫问题

```cpp
#include <iostream>
#include <cstring>

const int N = 110;
typedef std::pair<int, int> PII;

int n, m;
int g[N][N], d[N][N];
PII q[N], prev[N][N];

int bfs() {
	int hh = 0, tt = 0;
	q[0] = {0, 0};

	memset(d, -1, sizeof d);
	d[0][0] = 0;

	int dx[4] = {-1, 0, 1, 0}, dy[4] = {0, 1, 0, -1};
	while (hh <= tt) {
		auto t = q[hh ++];

		for (int i = 0; i < 4; i ++) {
			int x = t.first + dx[i], y = t.second + dy[i];
			if (x >= 0 && x < n && y >= 0 && y < m && d[x][y] == -1 && g[x][y] != 1) {
				d[x][y] = d[t.first][t.second] + 1;
				q[ ++ tt] = {x, y};
			}
		}
	}
	return d[n - 1][m - 1];
}

int main() {
	std::cin >> n >> m;
	for (int i = 0; i < n; i ++) {
		for (int j = 0; j < m; j ++) 
			std::cin >> g[i][j];
	}

	std::cout << bfs() << std::endl;

	return 0;
}

```
## 拓扑排序

```cpp
#include <iostream>
#include <cstring>

const int N = 100010;

int n, m;
int h[N], e[N], ne[N], idx;
int d[N];
int q[N];

void add(int a, int b) {
	e[idx] = b, ne[idx] = h[a], h[a] = idx ++;
}

bool topsort() {
	int hh = 0, tt = -1;
	for (int i = 1; i <= n; i ++) {
		if (d[i] == 0) {
			q[ ++ tt] = i;
		}
	}

	while (hh <= tt) {
		auto t = q[hh ++];

		for (int i = h[t]; i != -1; i = ne[i]) {
			int j = e[i];
			if (-- d[j] == 0) {
				q[ ++ tt] = j;
			}
		}
	}
	return tt == n - 1;
}

int main() {
	scanf("%d%d", &n, &m);

	memset(h, -1, sizeof h);

	while (m --) {
		int a, b;
		scanf("%d%d", &a, &b);
		add(a, b);
		d[b] ++;
	}

	if (!topsort()) puts("-1");
	else {
		for (int i = 0; i < n; i ++) printf("%d ", q[i]);
		puts("");
	}

	return 0;
}
```