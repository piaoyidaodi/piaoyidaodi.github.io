---
layout: post
title: "鸟哥的Linux私房菜--速查V"
categories: Linux
tag: Linux-Note
---
> 鸟哥的Linux私房菜——第二十章至第二十三章边读边记。

### 20. 开机流程、模块管理与Loader

#### 20.1 开机流程

主要开机流程大概如下：

按开机键

`-> BIOS`：获取CMOS中的BIOS配置，如硬件信息等。

`-> MBR`：硬盘第一个扇区446B的MBR，存储boot loader。

`-> boot loader`：每个系统含有各自独有的的boot loader。每一个filesystem或分区，都会保留一块启动扇区（boot sector）给bootloader，每个操作系统默认安装一套bootloader到自己的文件系统中。主要功能：**提供选单、加载内核、转交给其他loader**。

`--> 同时加载kernel`：一般被放置在/boot里，并取名为/boot/vmlinuz。磁盘、鼠标等作为模块核心功能（动态加载），存放在/lib/modules中。

`--> 同时加载initrd`：由于kernel需加载磁盘中的核心模块（包括磁盘驱动），而磁盘驱动就在磁盘中，存在矛盾。因此使用虚拟文件系统（Initial RAM Disk，一般存放在/boot/initrd中）。它可以被bootloader加载，并在内存中仿真一个根目录和一个可执行程序，加载核心模块。

`--> /sbin/init`：开始正常开机流程后的第一个程序（PID为1），主要用于准备软件执行环境，配置文件为/etc/inittab。inittab主要为runlevel，分为（0-6）级，详细man inittab。

`--> /etc/rc.d/rc.sysinit`：进行硬件信息初始化，可使用dmesg查看详细。基本所有的预设配置都在/etc/sysconfig中。如果想增加核心模块，将模块写入/etc/sysconfig/modules/*.modules中。

`--> /etc/rcN.d & /etc/sysconfig`：启动各项服务。根据runlevel设置N，执行对应/etc/rc.d/rcN.d目录下的脚本。其中对/etc/rcN.d/K??开头的文件执行stop，对/etc/rcN.d/K??执行start动作，所有的文件都是链接，该链接使用chkconfig进行处理。

`--> /etc/rc.d/rc.local`：任何开机时的自定义工作shell放入。

`--> 加载终端或X`：根据/etc/inittab的设定。

#### 20.2 开机过程中的主要配置文件

文件中的rc字母代表`runlevel change`。

`/etc/modprobe.conf`：进行自定义模块加载定义。

`/etc/sysconfig/*`
- autoconfig：规范使用者身份认证机制，默认使用MD5算法，并不使用外部身份验证机制。
- clock：设定Linux主机时区。
- i18n：语系。
- keyboard&mouse：设定键盘鼠标的形式。
- network：设定是否需启动网络，主机名和网关。

`init N`：切换run level；`runlevel`：查看run level。

#### 20.3 核心模块

核心模块放置在`/lib/modules/$(uname -r)/kernel`中，目录如下：
- `arch`：与硬件平台有关的项目，例如 CPU 的等级等等。
- `crypto`：内核所支持的加密的技术，例如md5或者是des等等。
- `drivers`：一些硬件的驱动程序，例如显示适配器、网络卡、PCI相关硬件等等。
- `fs`：内核所支持的filesystems，例如vfat, reiserfs, nfs等等。
- `lib`：一些函数库。
- `net`：与网络有关的各项协议数据，还有防火墙模块(net/ipv4/netfilter/*)等等。
- `sound`：与音效有关的各项模块。

使用`depmod [-Ane]`建立/lib/modules/$(uname -r)/modules.dep文件，记录核心模块相依性。
- `-A`：若加入该参数，则depmod会去搜寻更新模块，找到之后才会去更新。
- `-n`：不写入而是将结果输出到屏幕。
- `-e`：显示出目前已加载的不可执行模块名称。

`lsmod`显示模块名称、模块大小、此模块是否被其他模块使用。

`modinfo [-adln] [module_name|filename]`列出模块信息。
- `-a`：仅列出作者名称。
- `-d`：仅列出该modules的描述。
- `-l`：仅列出授权（license）。
- `-n`：仅列出该模块的详细路径。

`insmod [/full/path/module_name] [para]`：加载一个完整文件名模块，不主动分析模块相依性。不建议使用。

`rmmod [-fw] module_name`：不建议使用。
- `-f`：强制将该模块移除掉，不论是否正被使用。
- `-w`：若该模块正被使用，则rmmod会等待该模块被使用完毕后，才移除。

`modprobe [-lcfr] module_name`：自行处理加载问题。
- `-c`：列出目前系统所有的模块，更详细的代号对应表。
- `-l`：列出目前在`/lib/modules/$uname -r/kernel`当中的所有模块完整文件名。
- `-f`：强制加载该模块。
- `-r`：类似rmmod，就是移除某个模块。

#### 20.4 Grub

**grub分为两个阶段加载：**
- Stage1：执行MBR或boot sector中安装的最小主程序。
- Stage2：加载配置文件，一般在/boot/grub下，主要为menu.list或grub.conf配置文件，和各类文件系统定义。

**grub的特点：**
- 认识与支持较多的文件系统，并且可以使用grub的主程序直接在文件系统中搜寻内核；
- 开机的时候，可以自行编辑与修改开机设定项目，类似bash；
- 可以动态搜寻配置文件，而不需要在修改配置文件后重新安装grub。亦即是我们只需要修改完/boot/grub/menu.lst里的设定后，下次开机就生效了。

grub中的硬盘和分区，与Linux中的代号完全不同，如(hd0,0)，第一个搜到的硬盘号为0，第一个分区为0，并依此类推。

**/boot/grub/menu.lst配置文件：**
- title之前属于grub的整体设置，包括预设等待时间`timeout`与默认的开机项目`default`，还有显示的画面、显示选单`hiddenmenu`等特性。title 后面才是指定开机的内核或者是boot loader控制权。
- root指内核放置的分区目录。
- kernel指内核名。
- initrd指RAM Disk文件名。

**建立新的initrd文件：**

`mkinitrd [-v] [--with=模块名称] initrd 文件名 内核版本`：-v，显示运作过程。

**测试安装grub：**

`grub-install [--root-directory=DIR] install_device`：DIR为其他安装位置，默认为/boot/grub/XX；install_device为装置代号。

### 22. 软件安装：源码与Tarball

#### 22.1 分类与管理

RedHat系列（如 Fedora/CentOS 系列）使用RPM软件管理机制和yum在线更新模式；Debian使用的dpkg软件管理机制和APT在线更新模式。

#### 22.2 几类工具

**`ldconfig`与`/etc/ld.so.conf`**

`ldconfig [-f conf] [ -C cache] [-p]`：增加库文件的高速缓存
- -f conf：conf指的是某个文件名，即使用 conf 作为 libarary库的取得路径，而不以/etc/ld.so.conf作为默认值
- -C cache：cache指的是某个文件名，即使用 cache 作为快取暂存，而不以/etc/ld.so.cache为默认值
- -p ：列出目前有的所有凼式库资料内容（在 /etc/ld.so.cache 内的资料）

**`ldd`程序的动态链接库的解析**

`ldd [-vdr] [filename]`
- -v ：列出所有内容信息；
- -d ：重新将资料有遗失的link点秀出来
- -r ：将 ELF 有关的错误内容秀出来！

### 23. 软件安装：RPM，SRPM与YUM

#### 23.1 RPM与DPKG

**`dpkg`**：基于Debian，本地使用dpkg，在线使用apt；**`RPM`**：基于Red Hat，如Fedora，CentOS等，本地使用rpm/rpmbuild，在线使用yum。他们都提供了软件的依赖说明。

**SRPM**通常扩展名为XXX.src.rpm，提供源码文件，并比tar包多软件相依性说明。

#### 23.2 RPM的使用

**安装、升级**

`rpm -ivh [一个文件|多个文件|网络地址]`：-i是install的意思；-v查看详细安装信息；-h显示安装进度。

`rpm [--replacepkgs|--prefix path]`分别表示，安装时更新已安装的包，以及安装时安装到指定的目录。

`rpm [-Uvh|-Fvh] [一个文件|多个文件|网络地址]`：-U指软件若未安装过则直接安装，若已安装过则更新至最新版；-F指，不会安装未安装过的软件，只有安装过的软件会被升级。

**查询**

rpm在查询时，查询/var/lib/rpm/目录下的文件。
- `-q`：仅查询，后面接的软件名称是否有安装；
- `-qa`：列出所有的，已经安装在本机Linux系统上面的所有软件名称；
- `-qi`：列出该软件的详细信息（information），包含开发商、版本与说明等；
- `-ql`：列出该软件所有的档案与目录所在完整文件名（list）；
- `-qc`：列出该软件的所有配置文件（找出在 /etc/ 底下的文件而已)；
- `-qd`：列出该软件的所有说明文件（找出与man有关的档案而已）；
- `-qR`：列出与该软件有关的相依软件所含的档案（Required 的意思）
- `-qf`：由后面接的文件名，找出该档案属于哪一个已安装的软件；
- `-qp[icdlR]`：用途仅在于找出某个RPM档案内的信息，而非已安装的软件信息。

**卸载**

`rpm -e`：卸载。

`rpm --rebuilddb`：重建RPM文件库。

#### 23.3 YUM的使用

**查询**

`yum [-y|--installroot=path] [list|info|search|provides|whatprovides] [para]`进行查询。
- `-y`：提供自动的yes相应；
- `--installroot=/some/path`：将软件安装在指定path；
- `search`：搜寻某个软件名字或描述；
- `list`：列出yum所管理的所有软件名称与版本，类似于`rmp -qa`；
- `info`：同上，类似于`rmp -qai`；
- `provides`：类似于`rmp -qf`；

**安装、升级**

`yum [install|update]`。

**卸载**

`rpm remove`：卸载。