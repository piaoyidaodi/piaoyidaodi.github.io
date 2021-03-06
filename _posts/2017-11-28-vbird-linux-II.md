---
layout: post
title: "鸟哥的Linux私房菜--速查II"
categories: Linux
tag: Linux-Note
---
> 鸟哥的Linux私房菜——第九章至第十一章边读边记。

### 9. 文件的压缩与打包

#### 9.1 Linux常见压缩命令

1. 常见后缀名：
- `.Z`，compress程序压缩文件。已退出流行。
- `.gz`，gzip程序压缩的文件。
- `.bz2`，bzip2程序压缩的文件。
- `.tar`，tar程序打包的数据，没有经过压缩。
- `.tar.gz`，tar程序打包的文件，并进过gzip的压缩。
- `.tar.bz2`，tar程序打包的文件，并经过bz2的压缩。

2. `gzip`与`zcat`
- `gzip [-cdtv#] 文件名`。<br>
  `-c`将压缩数据输出到屏幕，可透过数据流重导向，如`gzip -c man.config > man.config.gz`；<br>
  `-d`解压缩；`-t`检查压缩文件的一致性；`-v`显示出文件的压缩比；<br>
  `-#`压缩等级，`-1`最快，压缩比最差，`-9`最慢压缩比最好，预设为`-6`。
- `gzip`压缩等级1-9,默认的6很好用。
- `zcat 文件名.gz`，`zcat`可读取纯文本文档被压缩后的压缩文件。

3. `bzip2`与`bzcat`
- `bzip2 [-cdkzv#] 文件名`；`bzcat 文件名.gz`。<br>
  `-c`将压缩数据输出到屏幕；`-d`解压缩；`-k`保留源文件；`-z`压缩；`-v`显示出文件的压缩比；<br>
  `-#`压缩等级，`-1`最快，压缩比最差，`-9`最慢压缩比最好，预设为`-6`。

#### 9.2 `tar`命令

1. `tar`<br>
- `tar [-j|-z] [cv] [-f 建立的文件名] filename`，打包与压缩。
- `tar [-j|-z] [tv] [-f 建立的文件名]`，查看文件名。
- `tar [-j|-z] [xv] [-f 建立的文件名] [-C 目录]`，解压缩。
  `-c`建立打包文件，可配合-v查看过程中打包的文件名；<br>
  `-t`查看打包文件里的内容；<br>
  `-x`解打包压缩功能，搭配-C 在特定目录解开，-c-t-x不能同时出现在一串指令中；<br>
  `-j`透过bzip2进行压缩或解压缩，此时文件名最好为*.tar.bz2；<br>
  `-z`透过gzip进行压缩或解压缩，此时文件名最好为*.tar.gz；<br>
  `-v`在压缩或解压缩过程中，将正在处理的文件名显示出来；<br>
  `-f filename`生成的目标文件名，建议单独写；<br>
  `-C 目录`，这个选项用在解压缩，解压到特定的目录中；<br>
  `-p`保留备份数据原本的权限和属性，常用于备份重要的配置文件。

2. 简单使用tar：
- 压缩，`tar -jcv -f filename.tar.bz2 要被压缩的文件或目录名称`；
- 查询，`tar -jtv -f filename.tar.bz2`;
- 解压缩，`tar -jxv filename.tar.bz2 -C 欲解压缩的目录`。

3. 仅仅打包的文件，称为tarfile。压缩打包的文件成为tarball。

4. `tar -cvf - 被打包目录 | tar -xvf -`，其中前后两个`-`代表标准输出和输入，`|`代表管线。此方法可以将一个目录拷贝至另一个目录。

#### 9.3 备份与恢复命令

1. `dump`备份整个文件系统或目录，但是对目录支持不足，仅支持对目录的完全备份。

2. `dump [-Suvj] [-level] [-f 目标文件] 源文件`，`dump -W`。
- `-S`，列出后面待备份数据需要多少磁盘空间才能够备份完毕。
- `-u`，将此次dump的时间记录到`/etc/dumpdates`文件中。
- `-v`，将dump的文件过程显示出来。
- `-j`，加入bzip2的支持，默认等级为2。
- `-level`，从`-0`到`-9`十个等级。
- `-f`，后接生成文件名。
- `-W`，列出在`/etc/fstab`中的具有`dump`设定的分区是否有备份过。

3. `restore`命令：
- `restore -t [-f dumpfile] [-h]`，`-t`查看dump文件，`-h`查看完整备份数据中的inode和文件系统信息。
- `restore -C [-f dumpfile] [-D 挂载点]`，`-C`与实际文件比较并列出与目前文件不一样的dump记录。
- `restore -i [-f dumpfile]`，进入交互模式可仅还原部分文件。
- `restore -r [-f dumpfile]`，用于还原整个文件系统

#### 9.4 光盘工具

1. `mkisofs [-o 映像文件][-rv][-m file] 待备份文件.. [-V vol] -graft-point isodir=systemdir ..`：建立映像文件。
- `-o`，后接目标映像文件名。
- `-r`，产生支持Unix/Linux的文件数据，可记录较多的信息。
- `-v`，显示创建ISO的过程。
- `-m file`，排除文件。
- `-V vol`，建立Volume，类似于Windows中的CD title。
- `-graft-point`，转移。

2. `cdrecord`：光盘刻录工具。

3. `dd`和`cpio`命令。

### 10. vim编辑器

#### 10.1 `vim`模式

1. vim 文档名，新建或打开已有文档。只有**一般模式**可以与**编辑、命令行模式切换**。
- **一般模式**：打开文档进入一般模式，可使用“上下左右”按键移动光标，可以使用删除来处理文档内容，也可复制粘贴。
- **编辑模式**：按下“i，I，o，O，a，A，r，R”等任何一个字母才进入编辑模式。若要退回到一般模式，需使用ESC键。
- **命令行模式**：在一般模式中，输入“：/?”任意一个进入命令行模式。

#### 10.2 `vim`常用按键说明

1. 在一般模式下：
- `G`移动到文件的最后一行；
- `gg`移动到文件的第一行；
- `n+<Enter>`光标下移n行；
- `/word`搜寻word这个字符串；
- `n`（此处为字母`n`）重复前一个搜寻动作；
- `:n1,n2s/word1/word2/g`，在第n1和n2行之间用word2替代word1；
- `:1,$s/word1/word2/g`，全篇寻找，并用word2替代word1；
- `:1,$s/word1/word2/gc`，全篇寻找，并用word2替代word1，取代前需用户确认；
- `x,X`，`x`为向后删除一个字符，`X`为向前删除一个字符；
- `dd`删除整行；`ndd`删除光标以下`n`列；
- `yy`复制游标所在行；`nyy`复制光标所在的下n列。
- `u`复原前一个动作；`ctrl+r`重做上一个动作；`.`重复前一个动作。

2. 一般模式切换编辑模式：
- `i,I`，进入**插入模式**，`i`为**从目前光标所在处插入**，`I`为**在目前所在行的第一个非空格符处开始插入**。
- `a,A`，进入**插入模式**，`a`为**在目前光标所在的下一个字符处插入**，`A`为**从光标所在行的最后一个字符处开始插入**。
- `o,O`，进入**插入模式**，`o`为**在目前光标所在的下一行处插入新的一行**，`O`为**在目前光标所在处的上一行插入新的一行**。
- `r,R`，进入**改写模式**，`r`为**只改写光标所在的那一个字符一次**，`R`为**一直改写，直到按下`ESC`键**

3. 一般模式切换命令行模式：
- `:w`写入硬盘；`:w!`强行写入，与用户对该文件的权限有关。
- `:q`离开vim；`:q!`强制离开不存储文件。
- `:wq`写入后存储；`:wq!`强制写入后存储，`:w 文件名`另存文档。
- `r 文件`将文件中的内容加到光标所在行的后面。
- `:set nu`设置显示行号；`:set nonu`取消行号。

#### 10.3 `vim`其他操作

1. `vim`会在与被编辑的档案目录下，再建一个名为`.filename.swp`的文件，所有的编辑操作会暂存在此文件中。

2. `:sp 文件名`，多窗口文档。`ctrl+W+上下键`，在窗口之间切换。

3. 文件换行符更改：
- `dos2unix [-kn] file [newfile]`
- `unix2dos [-kn] file [newfile]`
- `-k`:保留该文件原本的mtime；`-n`保留原文件，将转换后的输出到新文件。

### 11. 初识`Bash`

#### 11.1 初识`Shell`

1. `shell`用户操作系统的一个接口，通过壳程序操作其他应用程序，并与内核交互运作所需要的工作。

2. `bash`的优点：
- 各发行版本的bash是一致的；文字接口快，且不容易出现断线和信息外流问题。
- 命令编修能力（所谓的上下键就可以输入前后指令），放置在家目录的`.bash_history`中；
- 命令与文件补全功能（tab键）；
- 命令别名设定功能（alias）：`alias lm='ls -al'`可通过此种方式设置别名

3. `type`命令：
- `type [-tpa] name`查询指令是否是来源于外部或内建在bash中的指令。
  `-t`时name会以：`file`外部命令，`alias`，`builtin`指出命令含义；<br>
  `-p`如果后面接的name为外部指令，才会显示完整文件名；<br>
  `-a`列出所有name的信息。

#### 11.2 `Shell`变量功能

1. 环境变量通常以大写字母表示。

2. `echo`获取变量，变量前必须使用**`$`**符号，或**`${变量}`**的方式。

3. 变量设定规则：
- 改变变量使用等号，如`LANG=us_EN`。
- 其中`=`两边不能加空格；若变量之间有空格，则用双引号或单引号括住；单双引号的区别在于，**双引号仍可保留变量的内容，而单引号内仅能是一般字符，而不会有特殊符号**。
- 使用`export`使变量成为环境变量。
- 使用`unset`取消变量的设定。

#### 11.3 环境变量功能

1. `env`观察环境变量与常见环境变量说明；`set`观察所有变量（环境变量和自定义变量）。

2. `echo $$`本Shell的PID；`echo $?`返回上一个指令的错误码，无错误返回0。

3. 环境变量`OSTYPE，HOSTTYPE，MACHTYPE`，硬件和核心等级特点。

4. 子程序会继承父程序的环境变量，而不继承自定义变量。若想让子程序继承的环境变量，使用`export`变量名。

5. `locale`，查询Linux支持的语系。默认语系：`/etc/sysconfig/i18n`。

6. `read，array，declare`，使用键盘对变量读取、数组、与声明。
- `read [-pt] variable`，`-p`后面可以接提示字符；`-t`后面可以接等待的秒数。
- `declare [-aixr] varialbe`，`-a`将变量定义为数组；`-i`将变量定义为整型；`-x`将变量定义为环境变量；`-r`将变量定义为readonly类型，变量内容不可更改。
- `var[index]=content`，定义数组。

7. `ulimit`，与文件系统及程序的限制关系。

8. Linux下：clear清屏，ls列出文件和目录；windows下：cls清屏，dir列出文件和目录名。

#### 11.4 历史命令

1. `history`列举命令行中曾经下达过的指令；`history [n]`：列出最近的n笔命令行；`history [-c]`:删除目前shell中的所有history。

2. `history [-raw] histfiles`：
- `-a`，将目前新增的history指令新增到`histfiles`中，若没有加histfiles，则预设写入`~/.bash_history`。
- `-r`，将histfiles的内容读到目前这个shell的history记忆中。
- `-w`：将目前的history记忆内容写入histfiles中。

3. `$HISTSIZE`表明histfile的大小。

4. `!number`，执行第几个指令；`!command`，由最近的指令向前搜索指令开头为command的那个指令；`!!`，执行上一个指令，与上键效果一致。

#### 11.5 `Shell`操作环境

1. `bash`登陆时欢迎界面在`/etc/issue`中定义，代码含义：
- \d本地端时间的日期；
- \l显示第几个终端机接口；
- \m显示硬件等级；
- \n显示主机的网络名称；
- \o显示domain name；
- \r操作系统的版本；
- \t显示本地段时间的时间；
- \s操作系统名称；
- \v操作系统的版本。

2. `login shell`与`non-login shell`：
- `login shell`，取得bash时**需要**完整的登陆流程，如tty1-tty6。读取`/etc/profile`，系统文件；
- `non-login shell`，取得bash**不需要**重复登录，如X-Window登入后开启的终端，不需要再次输入帐号密码。
- `/etc/profile`为系统全局设定，建议不更改；`~/.bash_profile`或`~/.bash_login`或`~/.profile`，为个人设定，可更改写入。

3. `/etc/profile`和`~/.bash_profile`都是获取login shell时才会读取的配置文件，若需直接生效，使用`source`命令。

4. `~/.bashrc`在`non-login`登陆时读取（X-Window下打开终端）。

5. `stty [-a]`将目前所有的stty参数，按键与按键内容列出来。
- `eof`：`End of file`，代表输入结束，`Ctrl+D`；
- `erase`：向后删除字符，`Ctrl+?`；
- `intr`：送出一个中断信号给目前正在运行的程序，`Ctrl+C`；
- `stop`：停止当前的屏幕输出，`Ctrl+S`；
- `quit`：退出当前程序，`Ctrl+\`。

6. `set [-uvCHhmBx]`设置`bash`。

7. `bash`通配符：
- `*`代表“0到无穷多个”任意字符；
- `?`代表一个字符，如/?????，代表名称为五个字符的文件；
- `[]`含有方括号内的字符中的任何一个；
- `[-]`表示编码内的所有字符，如a-z；
- `[^]`，表示反向选取，如[^abc}代表一定含有一个非a,b,c的字符。

#### 11.6 数据流重定向

1. 数据流重导向：
- 以`>`方式输出，会覆盖原有的同名文件；
- 使用`>>`方式输出，会在存在文件的情况下，在文件的最下方累加上去；
- 如果输出的内容同时含有正确的和错误的信息，可以在之后使用两类导向符号，如 `>right 2>wrong`；
- 使用`2>dev/null`，将所有的错误数据导入虚空；
- 同时导入同一个文件，使用`2>&1 或 &>`，这样导出的文件数据比较规整，不会出现交叉的情况。

2. 定向：
- `1> `：以覆盖的方法将“正确的数据”输出到指定的文件或装置上；
- `1>>`：以累加的方法将“正确的数据”输出到指定的文件或装置上；
- `2> `：以覆盖的方法将“错误的数据”输出到指定的文件或装置上；
- `2>>`：以累加的方法将“错误的数据”输出到指定的文件或装置上。

#### 11.7 命令的执行判据

1. 顺序执行多个命令（不考虑命令间的相依性），命令之间使用`;`隔开。

2. 指令相依，使用&&和||连接相依命令：
- `cmd1 && cmd2`：<br>
  若 cmd1 执行完毕且正确执行($?=0)，则开始执行 cmd2。<br>
  若 cmd1 执行完毕且为错误($?≠0)，则 cmd2 不执行。
- `cmd1 || cmd2`：<br>
  若 cmd1 执行完毕且正确执行($?=0)，则 cmd2不执行。<br>
  若 cmd1 执行完毕且为错误 ($?≠0)，则开始执行 cmd2。

#### 11.8 各种命令

1. 管线命令(`pipe`)：
- 管线命令使用(**`|`**)作为界定符号；
- 仅能处理经由前面一个指令传来的正确信息，即`standard output`的信息，不能处理standard error。
- 管线后接的第一个数据必为命令，且可接受`std input`，如`less，more`等，而ls，cp则不是。

2. 撷取命令(`cut`)：处理以**行**为单位的信息。
- `cut -d'分割字符' -f fields`，`-d`：后面接分割字符与-f一起使用；`-f`：由`-d`分隔后，取出第几段。
- `cut -c 字符区间`，-c：以字符的单位取出固定字符区间。

3. 撷取命令(`grep`)：`grep [-acinv] [--color=auto] '搜寻字符串' 'filename'`，分析一行信息，若有需要的，则将该行拿出来。
- `-a`:将binary文件以text文件方式搜寻数据；
- `-c`:计算找到'搜寻字符串'的次数；
- `-i`:忽略大小写不同；
- `-n`:顺便输出行号；
- `-v`:反向选择，即显示出没有'搜索字符串'内容的那一行；
- `--color=auto`:可以将找到的关键词部分加颜色。

4. `sort`命令，`sort [-fbMnrtuk] [file or stdin]`：
- `-f`:忽略大小写差异；`-b`:忽略最前面的空格部分；
- `-M`:以月份名字来排序；`-n`:使用纯数字进行排序；
- `-r`:反向排序；`-u`:就是uniq，相同数据中，仅出现一行代表；
- -t:分隔符，预设使用[tab]键来分隔；-k:以区间(field)来进行排序。

5. `uniq`命令，`uniq [-ic]`，`-i`忽略大小写；`-c`进行计数。

6. `wc`命令，`wc [-lwm]`，`-i`仅列出行，`-w`仅列出字，`-m`多少字符。

7. `tee [-a] file`，双向重定向。

8. `tr,col,join,paste,expand`，字符处理命令。

9. `split [-bl] file PREFIX`，对大文件进行分割。

10. `xargs [-0epn] command`。

11. `-`可以替代`stdin`和`stdout`，如`tar -cvf - /home | tar -xvf -`。
