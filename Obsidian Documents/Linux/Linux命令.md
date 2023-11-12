
## 查看socket信息

`netstat`  /  `ss` 
netstat性能不好，推荐ss命令。
```cpp
选项：
-l：只显示LISTEN状态的socket
-t：只显示tcp
-n：以数字形式显示ip和端口
-p：显示进程信息
输出内容：
socket 的状态（_State_）、接收队列（_Recv-Q_）、发送队列（_Send-Q_）、本地地址（_Local Address_）、远端地址（_Foreign Address_）、进程 PID 和进程名称（_PID/Program name_）等。
```

## Linux三剑客
### sed

stream-edior的缩写。**逐行的读取并处理文件**。
不会直接生效再文件里面，使用-i可以直接生效。
```cpp
选项：
-i ：insert行前新增
-a ：append行后新增
-d ：delete删除
-c ：copy整行修改
-s ：substitute替换某些字符。可以结合正则表达式。 's/new/old/' 使用正斜杠，包括新值旧值。此时会只替换一个new，在最后加个g（global）正则表达式，代表全局替换。
-p ：查print 打印 

-e ：expression后跟**脚本表达式**，会产生备份文件。可以省略此选项。脚本表达是最好使用单引号。脚本表达中不指定行号默认所有行都生效   如：'1i\a new line'
-n ：不打印sed缓存区里的数据。
-f : 参数为文件名。读取文件中的多个脚本表达式。
```