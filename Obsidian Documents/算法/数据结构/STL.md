系统申请1000个空间和申请1个空间，所用时间几乎一样。关键在于申请空间的次数。因此vector中的空间是倍增思想，减少申请空间的次数。

# string

substr(iterator， (optional)size);
string.c_str();

# queue

没有clear()
清空：q = queue

# priority_queue

默认为大根堆。
小根堆定义的两种方式：
- 存相反数
- priority_queue<int, vector, greater>

# deque

加强版的vector

# set

erase :
- 输入为数，删除等于这个数的所有。O(K + logn)
- 输入为迭代器
lower_bound() : 返回大于等于x的最小数的迭代器 / 
upper_bound()：返回大于x的最小数的迭代器。
# bitset

定义：bitset<1000> S;
~、&、|、^、<<、>>
count()：返回有多少个1
any():返回是否有1
none()：是否全为0
set()：全部置1
set(k, v)：s[k] = v;
reset()：全部置0；
flip()：取反~