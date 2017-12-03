---
layout: post
title: "鸟哥的Linux私房菜--速查I"
categories: Linux
tag: Linux-Note
---
> 鸟哥的Linux私房菜——第五章至第八章边读边记。

### 5. 首次登陆：

#### 5.1 基础指令

1. `echo $LANG`显示语言；`LANG=en_US` 更改语系,并且注销后失效。

2. `date`显示日期与时间（date +%Y/%m/%d）。

3. `cal` 显示日期的指令（指令后可键入年份等）。

4. `bc` 计算器（scale=number设定小数点后位数）。

5. `tab`接在一串指令的第一个字后为命令补全；接在一串指令的第二个字后时为文件补齐。

6. `ctrl+c`：停止正在运行的程序；ctrl+d，相当于`exit`。

#### 5.2 指令帮助

1. `man`命令中的数字：**1** 表示“用户在shell环境中可以操作的指令或可执行文件”；**5** 表示“配置文件或者某些文件的格式”；**8** 表示“系统管理员可用的管理指令”。
- `/string`向下搜寻string这个字符串；`?string`向上搜寻string这个字符串。
- `man -f string`看文件的详细描述（精确查找string内容）；`man -k string`（匹配含string的项目）。

2. `info`与`man`用法相似

#### 5.3 其他指令

1. `nano ?.txt`打开`?.txt`无论其是否存在；M指Alt键。

2. 切换用户：
    su+用户名（su root）

3. 关机：
- shutdown [-hr]（系统服务关闭后就关机/重启） time（+10分钟或20:25类型或now） '提示的话'；
- halt硬关机  shutdown软关机。
- init+数字；**0**代表关机，**3**代表纯文本模式，**5**代表含有图形接口模式，**6**代表重新启动。

### 6. Linux文件权限与目录配置：

#### 6.1 文件属性

1. `/etc/passwd`保存系统的帐号（一般使用者和root）；`/etc/shadow`保存个人密码；`/etc/group`保存群组信息。

2. `ls -al`列出所有文件的详细权限与属性（包括隐藏文件，及首字符为“.”的文件）；-h将文件容量以KB等表示出来；-d列出目录本身而非文件。

3. 权限的第一个字符，`d`代表是目录，`-`代表是文件，`l`代表是链接档，`b`表示装置文件里的可供储蓄的接口设备，`c`表示装置文件里的串行端口设备（鼠标键盘等一次性读取装置）。第一组为文件拥有者的权限；第二组为同群组的权限，第三组为其他非本组的权限。

4. 中文可能出现乱码，所以更改语言为`LANG=en_US`。

5. 对于目录，如果只有r而没有x权限，则无法进入。

#### 6.2 属性和权限更改

1. 更改组所属：chgrp [-R] group dirname/filename；（目录加上-R 进行递归更改权限）。

2. 更改文件所有者：chown [-R] owner dirname/filename；

3. 更改权限：chmod [-R] XYZ(ugoa) dirname/filename；（其中XYZ分别代表`user,group,others`的权限，r:4，w:2，x:1）

4. Linux下文件是否可执行，需要看是否具有x权限，与windows的扩展名方式不同。拥有x权限不代表可以被执行成功。**Linux下的扩展名，只是可以方便用来识别文件类型**。

5. 对于目录，`r`权限表示可看到目录下的内容（未进入该目录，如果其中成员无x权限，没有查询该文件权限的权限），`w`表示可对目录下文件名有异动，`x`表示成员是否可切换进入该目录。

6. 必须与`/`目录在一起的有：
- `/etc`：**配置文件**，建议其中无可执行文件；
- `/bin`：单人维护模式下还可以使用的**重要执行文件**；
- `/dev`：所需要的装置与接口设备都以**文档**形式存在此文件中；
- `/lib`：开机时会用到的库文件，以及`/bin`和`/sbin`下指令使用的库文件；**执行文件所需的函数库与核心所需的模块**；
- `/sbin`：重要的系统执行文件，开机过程中所需要的命令，包括`fdisk,fsck,ifconfig,init,mkfs`等。

7. 其他重要目录：
- `/boot`，开机会使用到的文件，包括Linux核心文件以及开机选单和配置文件。
- `/usr`，类似windows系统的\Windows\和\Program files\这两个目录的综合体。

### 7. 文件与目录管理：

#### 7.1 目录操作

1. `cd`代表change directory，更改目录。`cd ~`回到自己家目录；`cd ~ string`回到string家目录。
2. `pwd`代表print working directory，显示目前所在目录。pwd -P得到正确目录名，而不是链接路径。
3. `mkdir [-mp] 目录名`，-m配置文件的权限，-p如果不存在帮助建立递归目录，存在不影响。
  ```sh
  mkdir -p test1/test2/test3 #建立多层次目录；
  mkdir -m 711 test2
  ```
4. `rmdir [-p] 目录名`，rmdir只能删除空目录。全部删除使用rm -r 目录名，递归删除全部。
5. `echo $PATH`，查询PATH。使用`PATH=“$PATH”:/root`。

#### 7.2 文件与目录管理

1. `ls [-adhl] [-color={never,auto,always}]`，`-d`仅列出目录本身。

2. `cp [-aipr] 源文件 目标文件`，复制文件或目录。`-a`相当于`-pdr`，`-r`递归复制，`-p`复制文件属性，`-d`若源文件为链接文件，则复制链接文件属性而非文件本身，`-i`若目标文件存在则覆盖前先询问。

3. `rm [-fir] 文件或目录`，-f忽略不存在的文件，不会出现警告信息；-i互动模式；-r递归删除，可以跳过多次询问。

4. `mv [-fiu] 源文件 目标文件`，`-u`若目标文件已经存在且源文件比较新才会更新，`-f`不询问强制覆盖。

5. `basename 路径`，路径文件名；`dirname 路径`，文件父目录名。

#### 7.3 文件内容查询

1. `cat [-AbEnTv]`，`-A=-vET`；`-b`列出行号；`-n`打印出行号，连同空白行也显示。`cat=concatenate`，`tac`与`cat`完全相反，从最后一行向第一行显示。

2. `nl [-bnw] 文件：`
- `-b a`：空行也列出行号；`-b t`：空行不列出行号。
- `-n ln`：行号显示在屏幕最左方；`-n rn`：行号在自己字段最右方显示，且不加0；`-n rz`：加0。
- `-w`：行号字段占用的位数。

3. `cat,tac,nl`全部显示所有数据。

4. `more`：一页一页向后翻页；（不好用）

5. `less`：一页一页向前翻页，`/string`，向下搜寻string；`?string`，向上搜寻string；`n`重复上一个搜索，`N`反向的重复上一个搜索。

6. `head [-n number] 文件`，`-n 数字`，显示几行，数字为负数表示后面的不显示。

7. `tail`与`head`用法相似，数字为正数，表示列出数字行后的行数。

8. `od [-t TYPE] 文件`，读取二进制文件。TYPE，a：默认字符输出；c：使用ASCII字符输出；

9. `touch [-acdmt] 文件`：
- `modification time(mtime)`，文件内数据发生变化更新此时间；
- `status time(ctime)`，文件状态如权限属性变化时，更新此时间；
- `access time(atime)`，文件数据被读取则更改。

#### 7.4 权限

1. `umask`为预设权限，共四位，除去哪种权限，则输入对应数字，比如除去其他用户的写权限则为`umask 002`。文件默认权限属性为666，目录默认权限属性777。

2. `chattr`配置隐藏属性，`chattr [+-=] [ASacdistu] 文件或目录名称`，`lsattr`显示隐藏属性。
- S同步文件修改到磁盘；
- a该文件只能增加数据，而不能删除和修改数据，且只有root可设置该属性；
- i该文件不能被删除、改名、设定连接也无法写入或新增资料，对于系统安全作用大，只有root可设定；
- s表示对文件彻底删除。

3. `file`，检查文件的基本数据。如ASCII或data文件或binary，是否用到动态链接库（share library）。

#### 7.5 文件搜寻

1. `which [-a] command(可执行文件)`，`-a`将`PATH`目录中可找到的所有指令列出。

2. `whereis [-bmsu] 文件名或目录名`，在数据缓存中搜寻。
- `-b`：只找binary格式；
- `-m`：只找在说明文件manual路径下的文件；
- `-s`：只找source来源文件；
- `-u`：搜寻其他不在以上三个地方的文件。

3. `locate [-ir] keyword`。`-i`忽略大小写差异；`-r`可接正则表达式。
  > `locate`搜寻特别快，因为`locate`所搜寻的数据是由`/var/lib/mlocate/`中的数据提供的，但是该数据库默认每天只执行一次，因此新建立的文件无法搜寻到。该数据库可通过**`updatedb`**命令调用`/etc/updatedb.conf`配置文件实现更新。

4. `find [PATH] [option] [action]`，费硬盘。
- 与时间相关：<br>
  `-mtime n`：n为数字意义为在n天之前的一天内，被改动过的内容文件；<br>
  `-mtime +n`：列出在n天之前（不含n天本身）被更动过内容的文件名；<br>
  `-mtime -n`：列出在n天之内（含n天本身）被更动过内容的文件名；<br>
  `-newer file`：file为一个存在的文件，列出比file还要新的文件名。<br>
- 与使用者或组名有关：<br>
  `-uid n`，n为数字对应帐号ID；<br>
  `-gid n`，n为数字对应组名ID；<br>
  `-user name`，name为使用者帐号名，如drift；-group name，name为组名；<br>
  `-nouser`和`-nogroup`分别为不属于`/etc/passwd`和`/etc/group`的文件。<br>
- 与文件权限及名称有关：<br>
  `-name filename`：搜寻文件名为filename的文件；<br>
  `-size [+-] SIZE`：搜寻比SIZE大[+]或者小[-]的文件。SIZE的规格有c（byte）和k（1024byte）；<br>
  `-type TYPE`：类型有正规文件（f），装置文件（b，c），目录（d），连结档（l），socket（s），以及FIFO（p）等属性。<br>
  `-perm [+.-] mode`：若mode为[4755]，则+为只要包含mode的任一属性即显示；-为包括mode所有属性即显示；无则代表正好符合mode属性即显示。

### 8. 文件系统管理

#### 8.1 文件系统

1. 扇区为最小的物理存储单元：
- 扇区围成圆，为磁柱，磁柱为分割槽的最小单位，每个扇区为512Byte；
- 第一个扇区最为重要，里面含有，主要开机区（`Master boot record，MBR，446Byte`）及分割表（`partition table，64Byte`）；
- 分割表`64Byte`，可记录四个分区，成为主分区或延伸分区，延伸分区可分割为逻辑分区，**只有主分区和逻辑分区可格式化**。

2. inode和block都有编号：
- `superblock`，记录此filesystem的整体信息，包括inode/block的总量，使用量，剩余量及文件系统格式。
- `inode`，记录文件属性，一个文件占用一个inode，同时记录文件数据所在的block号码。
- `block`，实际记录文件的内容，若文件太大，占用多个block。

3. 碎片整理，因为文件写入的block太过离散了，将使文件的读取能效很差。

#### 8.2 `inode`

1. `inode`记录的文件数据至少有：
- rwx；own/grp；
- 文件容量；
- ctime；atime；mtime；
- 文件特性，如setUID；
- 该文件真正的内容（pointer）。

2. `inode`特点：
- inode的大小固定为128Byte，每个文件都只能占用一个inode。
- 文件系统能创建的文件数与**inode**数量有关。
- 系统读取文件时先找到inode。并分析其中权限与用户是否相符，若符合才能读取block内容。

3. `inode`记录block号码区域定义为12个直接，一个间接，一个双间接和一个三间接记录区。

4. `Superblock`记录整个文件系统信息：
- 记录block和inode的总量；
- 未使用和已使用的inode/block的数量；
- `block`和`inode`的大小（`block`为1,2,4K，inode为128Byte）
- filesystem的挂载时间、最近一次写入数据的时间、最近一次检验磁盘的时间等文件系统的相关信息，一个flag说明文件是否挂载。

5. Filesystem Description（文件系统描述说明）描述每个block group的开始与结束的block号码，以及说明每个区段（superblock，bitmap，inodemap，datablock）分别介于哪一个block号码之间。

6. `dumpe2fs [-bh] 装置文件名`，-b列出保留为坏轨的部分，-h仅列出superblock数据。

#### 8.3 目录树

1. ext2文件系统下，建立目录时，ext2会分配一个inode与至少一块block给该目录。
- inode记录该目录的相关权限和属性以及block；
- 而block记录这个目录下的文件名与该文件名占用的inode号码数据。正因为如此目录下文件名的改变，需要该目录的w权限，因为文件名保存在block中。

2. 将文件系统与目录树相结合的动作称为，挂载。挂载点一定是目录，该目录为进入该文件系统的入口。

3. `ls -l /lib/modules/$(uname -r)/kernel/fs`，显示linux支持的文件系统；cat /proc/filesystems，显示已加载进去内存中支持的文件系统。

4. linux VFS(Virtual Filesystem Switch)，通过VFS读取filesystem，VFS自动识别文件系统并读取需求文件。

#### 8.4 文件系统简单操作

1. `df`显示出文件系统的整体磁盘使用量；`du`评估文件系统的磁盘使用量。
- `df [-ahikHTm] 目录或文件`。`-h`以人们易阅读的格式显示；`-i`不用硬盘容量，用inode数量显示。
- `du [-ahskm] 文件或目录`。`-h`易阅读；`-s`列出总量，而不列出每个个别的目录占用容量。

2. hard link不同目录使用同一inode；不能跨越Filesystem，不能link目录。

3. `ln[-sf] 来源文件 目标文件`。`-s`为symbolic link，同windows快捷方式；`-f`若目标文件存在，则删除目标文件后再建立。

4. `fdisk [-l] 装置名称`；`-l`列出装置中所有的partition内容。partprobe 分区完成后强制刷新分区表。

5. mkfs（make filesystem），使用默认值进行格式化工具。
- `mkfs [-t 文件系统格式] 装置名`。
- `mke2fs [-b block大小] [-i block大小] [-L 标头（卷名）] [-cj] 装置`，自定义格式化工具。

6. fsck（file system check），fsck [-t 文件系统] [-ACay] 装置名称，-a自动修复检查到的所有问题扇区；-y与-a类似，某些文件系统仅支持-y；-C使用直方图显示当前进度。

7. `badblocks [-svw] 装置名`，检查坏轨。`-s`在屏幕上列出进度，`-v`可以在屏幕上看到进度。很少使用。

8. `mount [-a] [-l] [-t 文件系统] [-n] [-L Lable名] [-o额外选项] 装置名 挂载点`，mount挂载文件系统。单一目录不应该重复挂载多个文件系统；要作为挂载点的目录最好是空目录。
- `-a`依照配置文件/etc/fstab的数据挂载所有磁盘；
- `-l`会显示Lable名称；
- `-t`后跟文件系种类，以选择Linux支持的格式。

9. `/etc/filesystems`：系统指定的测试挂载文件系统类型。

10. `/proc/filesystems`：系统已加载的文件系统类型。

11. `/lib/modules/$(uname -r)/kernel/fs/`：系统支持的文件系统驱动程序。

12. `umount [-fn] 装置文件名或挂载点`。
- `-f`强制卸载，可用在NFS 文件系统无法读取的情况；
- `-n`不更新`/etc/mtab`的情况下卸载。

13. `mknod 装置名 [bcp] [Major] [Minor]`。b：设定为存储文件，如硬盘；c设定为输入设备，如鼠标键盘。

14. `/etc/fstab`：利用mount挂载时，将所有的选项与参数写入到此文件中。

15. `mount -o loop 镜像名`，挂载虚拟设备，类似虚拟光驱。

16. `parted [装置] [指令[参数]]`，分割容量大于2T的磁盘。
- 新增分割：`mkpart[primary|logical|extended] [ext3|vfat] 开始 结束`；
- 分割表：print；
- 删除分割：`rm [partition]`。
