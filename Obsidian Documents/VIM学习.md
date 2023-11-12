# 插入

插入：a （字符后插入）；i（字符前插入）；o （下一行插入）；A（行末插入） I（行首插入）； O(上一行插入)；
删除：^w ^h ^u
dd x dw=daw diw dt)：（delete to ')' ）d0 d$ number+x ：删除number次
修改：r c(功能类似d+插入，ct) ct" caw ... ) s(删除当前字符+进入插入模式) R(进入替换模式) S(删除一行+插入模式)
查询：搜索方向：/ ？ 上个/下个：n/N  */#(当前单词匹配)
# 模式切换

normal -> inert : a i o
insert -> nomal : ctrl + c esc
visual : v V（以行为单位开启可视模式）
# 移动

方向：h j k l
最后一行/第一行：shift + g/ G gg
单词移动：E/e W/w b/B 大写为空白符分割，小写为其他分隔符
## 单行移动
f+字符，使用分号逗号切换。 t+字符（字符的前个位置） F+字符
## 水平移动
   ^(第一个非空白字符)/0(第一个字符)行首，$（第一个字符）/g_(第一个非空白)行尾
# 页面移动
文件开头：gg。结尾：shift+g/G
页面开头/中间/结尾：H/M/L
移动页面而非光标，翻页：ctrl+u ctrl+f zz
# 多文件操作
## 缓冲区

ls列出缓冲区，b+n进行切换。
## 窗口

sp垂直分割，vs水平分割。也可以ctrl+w / ctrl + v
切换：ctrl + w/h/j/k/l。
窗口大小设置：
## 标签
# vim text object

三种object：**word，sentence, paragraph** 

正常模式：number + command + text objet
iw : inner word, aw : around word , ip/ap
例如：5daw：删除5个单词包括空格。
![](Pasted%20image%2020231028012848.png)
例如：di{
# vim复制粘贴和寄存器
## 复制粘贴

normal模式下的复制粘贴：
- y(yank) p(put)
- v进入可视模式之后按y进行复制
- yiw，yaw复制单词，yy复制一行

> insert模式复制的问题

编码时经常会设置autoindent，此时再复制代码会导致缩进混乱。
解决：`set paste`粘贴完后再`set nopaste`
## 寄存器

vim里的操作默认放到无名寄存器而不是系统剪贴板。
- 使用指定寄存器：normal模式，`"{register}`。如normal模式下使用`"ayiw`可以复制单词到a寄存器。
- 查看指定寄存器的内容：末行模式，`:{register}`
### 寄存器种类

- a~z寄存器
- "：无名寄存器
- 0：复制专用寄存器
- +：系统剪贴板，需要`has('clipborad') == 1`。默认使用系统剪贴板：`set clipboard=unnamed`
- `"%`：当前文件名；`".`：上次插入的文本。
# macro

使用场景：如给一个文件中的所有url加双引号；进行批量编辑；进行重复操作时。
定义：一系列命令的集合。可以进行录制和回放。
命令：q 录制/结束录制。录制会放到寄存器中。可以使用`q{register}`指定录制寄存器，使用`@{register}`指定回放寄存器。可以结合末行模式进行选中回放。
# 补全

![300](Pasted%20image%2020231110225929.png)
常用的三种：
![](Pasted%20image%2020231110230101.png)
补全选项选择：^n表示next，^p表示上个。
