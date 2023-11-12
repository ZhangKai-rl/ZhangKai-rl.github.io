要在编译的时候加上-g选项

gdb + 可执行文件进入调试

```c++
启动调试：
	gdb 文件名
设置此可执行文件的命令行参数
	set args 参数值
查看参数
	show args
查看代码：
	list
	list 文件名：行号/函数名
查看当前所在文件
	info source
往下执行：
	往下执行一行：
		next：遇见函数不进入
		step：步进  进入执行的函数
	**执行到下个断点**
		continue
开始进行调试，停在断点处：
	run / start
打印当前文件下的变量值：
	print
	p &变量名   // 打印变量地址
监视变量（变化时打印）
	watch *变量地址
每次执行都打印变量的值（自动变量，常用于循环中）
	display 变量名
	i display 查看自动变量
	undisplay 变量名 取消自动变量
设置变量值 set var 变量名 = 变量值
打断点：
	break 文件名：行号
设置断点条件：（用于循环中）  // 将循环中的值改为这个
	b 行号 if 循环变量=值
在某次循环设置断点
	1. 先设置断点: b 行号
	2. condition 断点号 循环变量 == 某个值
跳过for循环：
	until
立即执行完当前函数：
	finish
查看函数栈、查看当前文件及位置：
	where
设置断点有效/无效/删除断点：
	enable/disable 断点号
	delete 断点号  // 不加删除所有断点
设置临时断点 
	until 行号
打印当前的函数调用栈的所有信息:
	backtrace
	where
查看当前栈帧的信息
	frame
移动查看其它栈帧的信息：
	up/down
查看core文件
	1. 使用ulimit -c开启core文件

```

多线程命令：
```shell
(1)查看可切换调试的线程：info threads
(2)切换调试的线程：thread 线程id
(3)只运行当前线程：set scheduler-locking on
(4)运行全部的线程：set scheduler-locking off
(5)指定某线程执行某gdb命令：thread apply 线程id gdb_cmd
(6)全部的线程执行某gdb命令：thread apply all gdb_cmd
```