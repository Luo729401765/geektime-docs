你好，我是LMOS。

工欲善其事，必先利其器。作为开发者，学习过程中我们尤其要重视动手实践，不断巩固和验证自己学到的知识点。而动手实践的前提，就是要建立一个开发环境，这个环境具体包括编译环境、执行环境，以及各种常用的工具软件。

我会用两节课带你动手搭好环境，今天这节课咱们先热个身，搞清楚什么是主环境，还有怎么基于它生成交叉编译工具。

代码你可以从[这里](https://gitee.com/lmos/Geek-time-computer-foundation/tree/master/lesson12~13)下载。

## 主环境

主环境，有时也叫作HOST环境，也就是我们使用的计算机环境，即使用什么样的操作系统、什么架构的计算机作为开发环境。

比方说我们经常用PC机作为开发机使用，它实际就是一个基于x86架构（或其他架构）的硬件平台，再加上Windows或者Linux等操作系统共同组成的开发环境。

普通用户的电脑上经常安装的操作系统是Windows，因为界面友好方便、操作简单且娱乐影音、游戏办公等应用软件也是不胜枚举。

Windows对普通用户来说的确非常友好。但是作为软件开发者，对于志存高远、想要精研技术的我们而言，更喜欢用的是Linux系统。

它虽然没有漂亮的GUI，却暴露了更多的计算机底层接口，也生产了更多的开发工具和各种各样的工具软件。比如大名鼎鼎的编译器GCC、声名远扬的编辑器EMACS、VIM，还有自动化的脚本工具shell、make等。这些工具对开发者非常友好，配合使用可以让我们的工作事半功倍，后面你会逐渐体会到这点。

当然Linux只是一个内核，我们不能直接使用，还需要各种工具、库和桌面GUI，把这些和Linux打包在一起发行，这就构成了我们常说的Linux发行版。

我最喜欢的Linux发行版是Deepin和Ubuntu。为了统一，我建议你使用Deepin最新版，你也可以使用Ubuntu，它们是差不多的。只是操作界面稍有不同。我先给你展示下我的Deepin，如下图，刚装上它的时候，我就觉得它颇为惊艳。

![图片](https://static001.geekbang.org/resource/image/78/28/782c1d620430ba46b584b0cdbd86dc28.jpg?wh=1920x1089)

这里最基础的安装我就不讲了，因为安装Deepin十分简单，无论是虚拟机还是在物理机上安装，我相信你通过互联网都可以自行解决，搞不定也可以看看[这里](https://www.deepin.org/zh/installation/)。

这两种方式我也替你对比过，虚拟机中的Linux较物理机上的Linux性能稍差一点，但并不影响我们实验操作和结果。

## 为什么需要交叉编译

虽然主环境搞定了，但现在我们还不能直接跑代码。为什么呢？

先回想一下，平时我们正常开发软件需要什么？我猜，哪怕你不能抢答，也会知道个大概：需要电脑（PC）、特定的操作系统（比如Windows或Linux等），在这个操作系统上还能运行相应的编辑器和编译器。编辑器用来编写源代码，而编译器用来把源代码编译成可执行程序。

似乎不需要更多东西了，毕竟我们日常开发的软件，宿主平台和目标平台是相同的。如果我们把限制条件变一变，情况就不同了。如果我们想尝试在RISC-V平台上跑程序，要怎么办呢？

你或许会说，这简单，买一台RISC-V的机器不就行了。可是先不说购买硬件的经济成本，实际上，很多RISC-V平台硬件资源（如内存、SD卡容量）有限，不足以运行复杂的编译器软件，有的甚至没有操作系统，更别说在上面运行编译器或者编辑器软件了。

面对这样的困境，就要用到**交叉编译**了。什么是交叉编译呢？简单来说，就是在一个硬件平台上，生成另一个硬件平台的可执行程序。

举个例子，我们在x86平台上编译生成ARM平台的可执行程序；再比如说，之后的课里我们将在x86平台上，生成RISC-V的可执行程序。这些都属于交叉编译，在这个过程中编译生成可执行程序的平台，称为宿主机或者主机；执行特定程序的平台（如ARM或者RISC-V平台），称为目标机。

我特意准备了图解，为你展示在x86平台上，交叉编译生成RISC-V平台可执行程序的过程，你可以仔细看看：

![图片](https://static001.geekbang.org/resource/image/a5/55/a5fb7d81fa49da02bfeee45c35aef255.jpg?wh=1920x1183)

## 如何构建RISC-V交叉编译器

前面说了交叉编译的本质就是生成其他平台体系上的可执行程序，这个体系又不同于我们宿主平台。我们的目的很简单，就是要**在x86平台上编写源代码，然后编译出RISC-V平台的可执行程序，最后放在RISC-V平台上去运行**。

因此，我们需要用宿主机编译器A，编译出一个编译器B，这个编译器B是本地平台上的可执行程序。

说得再具体点，你可以把编译器B看作是 **x86 Linux上的一个应用**。但它的特殊之处就是，能根据源代码生成RISC-V平台上的可执行程序。补充一句，这里的编译器A和B都是C语言编译器。

下面我们开始构造编译器B。编译器B不仅仅是C语言编译器，还有很多额外的程序。比如RISC-V平台上使用的二进制文件分析（objcopy）、反汇编（objdump）、elf结构分析工具（readelf）、静态库归档（ar）、汇编器（as）、链接器（ld）、GDB、C语言库（Newlib、Glib、Multlib）等。

为了简单、便于区分，我们把这些对应于RISC-V平台的编译器相关的软件，统称为 **RISC-V工具链**。

构建RISC-V工具链的主要步骤如下：

1.安装依赖工具：在宿主平台上安装编译器A，以及相应的工具和库。  
2.下载RISC-V工具链的源代码；  
3.配置RISC-V工具链；  
4.编译RISC-V工具链，并安装在宿主平台上。

### 第一步：安装依赖工具

我们先从第一步开始，编译器A主要是宿主平台上的GCC，工具主要是Make、Git、Autoconf、Automake、CURL、Python3、Bison、Flex等。这里GCC主要在build-essential包中，我们只要在Linux终端中输入如下指令就可以了：

```plain
sudo apt-get install git autoconf automake autotools-dev curl python3 libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf patchutils bc libexpat-dev libglib2.0-dev ninja-build zlib1g-dev pkg-config libboost-all-dev libtool libssl-dev libpixman-1-dev libpython-dev virtualenv libmount-dev libsdl2-dev
```

![图片](https://static001.geekbang.org/resource/image/77/79/778c6aea3fa8c112c732fc3d7b653f79.jpg?wh=1910x1154)

如果不出意外，这些工具和库会通过网络由Linux的apt包管理器，全自动地给你安装完毕。

### 第二步：下载工具链源代码

接着进入第二步下载RISC-V工具链源代码。通常来说，我们只要用Git克隆一个riscv-gnu-toolchain仓库即可，其它的由riscv-gnu-toolchain仓库中的仓库子模块自动处理。

#### 手动配置环节

由于众所周知的网络原因，你可能连riscv-gnu-toolchain仓库都下载不下来，更别说自动下载仓库子模块了。为了照顾卡壳的人，我把手动处理的情况也顺便讲一下，能够直接自动安装的同学可以跳过这部分，直接翻到7条指令之后的最终截图对一下结果就行。

子模块如下：

```plain
riscv-qemu（虚拟机）
riscv-newlib (用嵌入式的轻量级C库)
riscv-binutils(包含一些二进制工具集合，如objcopy等)
riscv-gdb(用于调试代码的调试器)
riscv-dejagnu(用于测试其它程序的框架)
riscv-glibc(GNU的C库)
riscv-gcc (C语言编译器)
```

这些子模块我们需要手动从Gitee网站上下载。下载前，我们先在终端上输入后面的指令，建立一个目录，并切换到该目录中：

```plain
mkdir RISCV_TOOLS
cd RISCV_TOOLS
```

把RISC-V工具链的源代码手动下载好，步骤稍微多了一些，我在后面分步骤列出，方便你跟上节奏。

其实也就是7条指令的事儿，并不复杂。先统一说明下，**后面这些命令都是切换到riscv-gnu-toolchain目录的终端下**，输入我给你列出的指令即可。

1.开始下载riscv-gnu-toolchain，命令如下：

```plain
git clone https://gitee.com/mirrors/riscv-gnu-toolchain
cd riscv-gnu-toolchain
```

2.下载RISC-V平台的C语言编译器源代码仓库，输入如下指令：

```plain
git clone -b riscv-gcc-10.2.0 https://gitee.com/mirrors/riscv-gcc
```

3.下载测试框架源代码仓库，即riscv-dejagnu。输入如下指令：

```plain
git clone https://gitee.com/mirrors/riscv-dejagnu
```

4.下载GNU的C库源代码仓库，也就是riscv-glibc，输入如下指令：

```plain
git clone -b riscv-glibc-2.29 https://gitee.com/mirrors/riscv-glibc
```

5.下载用于嵌入式的轻量级C库源代码仓库，即riscv-newlib。输入如下指令：

```plain
git clone https://gitee.com/mirrors/riscv-newlib
```

6.下载二进制工具集合源代码仓库riscv-binutils，输入如下指令：

```plain
git clone -b riscv-binutils-2.35 https://gitee.com/mirrors/riscv-binutils-gdb riscv-binutils
```

7.最后，下载GDB软件调试器源代码仓库riscv-gdb，输入如下指令：

```plain
git clone -b fsf-gdb-10.1-with-sim https://gitee.com/mirrors/riscv-binutils-gdb riscv-gdb
```

好，现在所有的RISC-V工具链的源代码，我们已经下载完了。我们一起来同步一下，确保你我的riscv-gnu-toolchain目录下的目录和文件，完全一致。

在riscv-gnu-toolchain目录的终端下输入ls指令，你应该得到和后面这张图一样的结果。

![图片](https://static001.geekbang.org/resource/image/e8/34/e8307c6cabf00a3fccc4a0e0a93b3034.jpg?wh=1910x1154)

### 第三步：配置工具链

在我们用宿主编译器编译所有的RISC-V工具链的源代码之前，还有最重要的一步，那就是配置RISC-V工具链的功能。

RISC-V工具链有很多配置选项，不同的配置操作会生成具有特定功能的RISC-V工具链。此外，配置操作还有一个功能，就是检查编译RISC-V工具链所依赖的工具和库。检查通过，就会生成相应的配置选项文件，还有用于编译操作的Makefile文件。

下面我们开始配置操作。为了不污染源代码目录，我们可以在riscv-gnu-toolchain目录下建立一个build目录，用于存放编译RISC-V工具链所产生的文件。还是在切换到riscv-gnu-toolchain目录的终端下，输入如下指令：

```plain
mkdir build  #建立build目录
#配置操作，终端一定要切换到build目录下再执行如下指令
../configure --prefix=/opt/riscv/gcc --enable-multilib --target=riscv64-multlib-elf
```

我给你解释一下指令里的关键内容。

–prefix表示RISC-V的工具链的安装目录，我们一起约定为“/opt/riscv/gcc”这个目录。

–enable-multilib表示使用multlib库，使用该库编译出的RISC-V工具链，既可以生成RISCV32的可执行程序，也可以生成RISCV64的可执行程序，而默认的Newlib库则不行，它只能生成RISCV（32/64）其中之一的可执行程序。

–target表示生成的RISC-V工具链中，软件名称的前缀是riscv64-multlib-elf-xxxx。若配置操作执行成功了，build目录中会出现如下所示的文件：

![图片](https://static001.geekbang.org/resource/image/86/5e/86a3b65f445394f1e8be61e3470e9f5e.jpg?wh=1920x1271)

### 第四步：编译工具链

最后我们来完成第四步，编译RISC-V工具链。只要配置操作成功了，就已经成功了90%。其实编译操作是简单且高度自动化的，我们只要在切换到build目录的终端下，输入如下指令即可：

```plain
sudo make -j8
```

这个指令在编译完成后会自动安装到“/opt/riscv/gcc”目录，由于要操作“/opt/riscv/gcc”目录需要超级管理员权限，所以我们要记得加上sudo。

另外，如果你的宿主机的CPU有n个核心，就在make 后面加-j（n\*2），这样才能使用多线程加速编译。

好了，一通操作猛如虎，现在最重要的事情是等待计算机“搬砖”了。你不妨播放音乐，泡上一杯新鲜的热茶，一边听歌，一边喝茶……估计要喝很多杯茶，才会编译完成。最最重要的是这期间不能断电，否则几个小时就白费了。

如果终端中不出现任何错误，就说明编译成功了。我们在终端中切换到“/opt/riscv/gcc/bin”目录下，执行如下指令：

```plain
riscv64-unknown-elf-gcc -v
```

上述指令执行以后，会输出riscv64-unknown-elf-gcc的版本信息，这证明RISC-V工具链构建成功了。如下所示：

![图片](https://static001.geekbang.org/resource/image/d6/be/d62cf1b12d532aea1643118ea117febe.jpg?wh=1920x1271)

到这里，我们环境已经成功了一半，有了交叉编译器，并且这种交叉编译器能生成32位的RISC-V平台的可执行程序，也能生成64位的RISC-V平台的可执行程序。

你可能会好奇，成功了一半，那另一半呢？这需要我们接着干另一件事。什么事呢？容我先在这里卖个关子，下节课再揭秘。

## 重点回顾

通过这节课的学习，我们成功构建了RISC-V工具链，这样就能在X86平台上生成RISC-V平台的可执行程序了。下面让我们一起回顾一下，这节课中都做了些什么。

我们首先约定了宿主环境，需要用到Ubuntu或者Deepin的Linux发行版，无论你是将它们安装在物理PC上，还是安装在虚拟机上。

然后我们了解了什么是交叉编译。为了方便后面课程学习动手实践，我们要在x86平台的宿主机上编译生成RISC-V平台的可执行程序。

明确了目标，我们一起动手开始构建了一个RISC-V交叉编译器。你会发现其中不只是C/C++编译器，还有很多处理二进制可执行程序的工具，我们把这些统称为RISC-V工具链。

## 思考题

请你说一说交叉编译的过程？

期待你再留言区分享自己的实验笔记，或者与我交流讨论。也推荐你把这节课分享给更多朋友，我们一起玩转交叉编译。
<div><strong>精选留言（15）</strong></div><ul>
<li><span>Vic</span> 👍（5） 💬（3）<p>host使用的是Ubuntu 20.04 Desktop, 在Windows 11的笔记本, 用Virtualbox 6.1 搭建的虚机 (cpu 2 cores, 40GB 存储，4GB内存) ，按照老师的教程，以下是我的实验笔记： 
1. 在我的环境就只有一个libpython-dev 报错，因为有提示换成libpython2-dev，不碍事，而且不同的os，不同的版本，可能不会有这个问题，可以先用老师的命令。
sudo apt-get install git autoconf automake autotools-dev curl python3 libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf patchutils bc libexpat-dev libglib2.0-dev ninja-build zlib1g-dev pkg-config libboost-all-dev libtool libssl-dev libpixman-1-dev libpython2-dev virtualenv libmount-dev libsdl2-dev
2. mkdir RISCV_TOOLS; cd RISCV_TOOLS
3.
git clone https:&#47;&#47;gitee.com&#47;mirrors&#47;riscv-gnu-toolchain
cd riscv-gnu-toolchain
4.
mkdir build 
cd build
..&#47;configure --prefix=&#47;opt&#47;riscv&#47;gcc --enable-multilib --target=riscv64-multlib-elf
5. 因为我的host环境是建在2cpu core的虚机上
sudo make -j 2 
6. 大约3个多小时，终于完成了。
7. cd &#47;opt&#47;riscv&#47;gcc&#47;bin
.&#47;riscv64-unknown-elf-gcc -v
8. 结果输出：
Using built-in specs.
COLLECT_GCC=.&#47;riscv64-unknown-elf-gcc
COLLECT_LTO_WRAPPER=&#47;opt&#47;riscv&#47;gcc&#47;libexec&#47;gcc&#47;riscv64-unknown-elf&#47;12.1.0&#47;lto-wrapper
Target: riscv64-unknown-elf
Configured with: &#47;home&#47;vic&#47;RISCV_TOOLS&#47;riscv-gnu-toolchain&#47;build&#47;..&#47;gcc&#47;configure --target=riscv64-unknown-elf --prefix=&#47;opt&#47;riscv&#47;gcc --disable-shared --disable-threads --enable-languages=c,c++ --with-pkgversion= --with-system-zlib --enable-tls --with-newlib --with-sysroot=&#47;opt&#47;riscv&#47;gcc&#47;riscv64-unknown-elf --with-native-system-header-dir=&#47;include --disable-libmudflap --disable-libssp --disable-libquadmath --disable-libgomp --disable-nls --disable-tm-clone-registry --src=&#47;home&#47;vic&#47;RISCV_TOOLS&#47;riscv-gnu-toolchain&#47;gcc --enable-multilib --with-abi=lp64d --with-arch=rv64imafdc --with-tune=rocket --with-isa-spec=2.2 &#39;CFLAGS_FOR_TARGET=-Os   -mcmodel=medlow&#39; &#39;CXXFLAGS_FOR_TARGET=-Os   -mcmodel=medlow&#39;
Thread model: single
Supported LTO compression algorithms: zlib
gcc version 12.1.0 () 
9. 收工。
</p>2022-08-22</li><br/><li><span>光华路小霸王</span> 👍（5） 💬（3）<p>编译中会卡在 
Cloning into &#39;&#47;home&#47;qing&#47;RISCV_TOOLS&#47;riscv-gnu-toolchain&#47;gcc&#39;...
还是会去下载仓库，应该是我们下载的文件夹名称不是默认的，查看 README.md 文件，在最后说明了如何指定子模块的路径，在配置环节添加配置之后，戴尔笔记本双核  i5-4200U CPU @ 1.60GHz 编译，编译完成四十多分钟，系统使用  Debian 4.19.181-1 

..&#47;configure --prefix=&#47;opt&#47;riscv&#47;gcc \
--enable-multilib \
--target=riscv64-multlib-elf \
--with-gcc-src=&#47;home&#47;name&#47;RISCV_TOOLS&#47;riscv-gnu-toolchain&#47;riscv-gcc \
--with-binutils-src=&#47;home&#47;name&#47;RISCV_TOOLS&#47;riscv-gnu-toolchain&#47;riscv-binutils \
--with-newlib-src=&#47;home&#47;name&#47;RISCV_TOOLS&#47;riscv-gnu-toolchain&#47;riscv-newlib \
--with-glibc-src=&#47;home&#47;name&#47;RISCV_TOOLS&#47;riscv-gnu-toolchain&#47;riscv-glibc \
--with-gdb-src=&#47;home&#47;name&#47;RISCV_TOOLS&#47;riscv-gnu-toolchain&#47;riscv-gdb
</p>2022-08-27</li><br/><li><span>筱琲</span> 👍（1） 💬（2）<p>用虚拟机的需要注意下，CPU核数，内存，磁盘，都要尽可能大一些，我设置的是4核&#47;8G&#47;100G，这样才能一路通关。不然要么卡死要么到最后提示空间不足，你连系统都启动不了。</p>2022-11-02</li><br/><li><span>bubble</span> 👍（1） 💬（2）<p>mkdir -p stamps&#47; &amp;&amp; touch stamps&#47;build-gdb-newlib 卡住一直不动了。
 riscv-gnu-toolchain git:(master) ✗ ps -aux | grep make
root      580153  0.1  0.0  19764  4876 pts&#47;1    S+   15:36   0:00 sudo make -j8
root      580156  0.0  0.0  19764   748 pts&#47;2    Ss   15:36   0:00 sudo make -j8
root      580157  0.0  0.0  11792  2356 pts&#47;2    S+   15:36   0:00 make -j8
paralle+  621310  0.0  0.0  12120  1844 pts&#47;3    S+   15:38   0:00 grep --color=auto --exclude-dir=.bzr --exclude-dir=CVS --exclude-dir=.git --exclude-dir=.hg --exclude-dir=.svn --exclude-dir=.idea --exclude-dir=.tox make</p>2022-09-01</li><br/><li><span>吴卫</span> 👍（0） 💬（1）<p>你好，请问你当时使用的riscv-gnu-toolchain是什么版本的？我刚订了你的课程，由于riscv-gnu-toolchain版本更新，所以按照当前的课程指导装不起来。
</p>2024-10-16</li><br/><li><span>gzh4869</span> 👍（0） 💬（1）<p>直接sudo apt install gcc-riscv64-unknown-elf 是不是也行</p>2023-01-11</li><br/><li><span>Geekysl17</span> 👍（0） 💬（1）<p>请问我在ubuntu20上运行sudo make -j8后为什么出现下面的问题啊，怎么解决呢？
Submodule &#39;gcc&#39; (https:&#47;&#47;gcc.gnu.org&#47;git&#47;gcc.git) registered for path &#39;gcc&#39;
Submodule &#39;binutils&#39; (https:&#47;&#47;sourceware.org&#47;git&#47;binutils-gdb.git) registered for path &#39;binutils&#39;
Cloning into &#39;&#47;home&#47;ysl&#47;code&#47;OS&#47;RISCV_TOOLS&#47;riscv-gnu-toolchain&#47;gcc&#39;...
Submodule &#39;newlib&#39; (https:&#47;&#47;sourceware.org&#47;git&#47;newlib-cygwin.git) registered for path &#39;newlib&#39;
Cloning into &#39;&#47;home&#47;ysl&#47;code&#47;OS&#47;RISCV_TOOLS&#47;riscv-gnu-toolchain&#47;binutils&#39;...
Cloning into &#39;&#47;home&#47;ysl&#47;code&#47;OS&#47;RISCV_TOOLS&#47;riscv-gnu-toolchain&#47;newlib&#39;...
Submodule &#39;gdb&#39; (https:&#47;&#47;sourceware.org&#47;git&#47;binutils-gdb.git) registered for path &#39;gdb&#39;
Cloning into &#39;&#47;home&#47;ysl&#47;code&#47;OS&#47;RISCV_TOOLS&#47;riscv-gnu-toolchain&#47;gdb&#39;...
fatal: unable to access &#39;https:&#47;&#47;sourceware.org&#47;git&#47;binutils-gdb.git&#47;&#39;: gnutls_handshake() failed: Error in the pull function.
fatal: clone of &#39;https:&#47;&#47;sourceware.org&#47;git&#47;binutils-gdb.git&#39; into submodule path &#39;&#47;home&#47;ysl&#47;code&#47;OS&#47;RISCV_TOOLS&#47;riscv-gnu-toolchain&#47;binutils&#39; failed
Failed to clone &#39;binutils&#39;. Retry scheduled
</p>2022-12-05</li><br/><li><span>miraclezhb</span> 👍（0） 💬（1）<p>&#47;home&#47;miraclezhb&#47;projects&#47;RISCV_TOOLS&#47;riscv-gnu-toolchain&#47;build&#47;..&#47;riscv-gdb&#47;gdb&#47;ada-exp.y: In function ‘int ada_parse(parser_state*)’:
&#47;home&#47;miraclezhb&#47;projects&#47;RISCV_TOOLS&#47;riscv-gnu-toolchain&#47;build&#47;..&#47;riscv-gdb&#47;gdb&#47;ada-exp.y:736:15: error: ‘yyin’ was not declared in this scope; did you mean ‘yyrline’?
  736 |   lexer_init (yyin);  &#47;* (Re-)initialize lexer.  *&#47;
      |               ^~~~
      |               yyrline
&#47;home&#47;miraclezhb&#47;projects&#47;RISCV_TOOLS&#47;riscv-gnu-toolchain&#47;build&#47;..&#47;riscv-gdb&#47;gdb&#47;ada-exp.y:736:3: error: ‘lexer_init’ was not declared in this scope; did you mean ‘pex_init’?
  736 |   lexer_init (yyin);  &#47;* (Re-)initialize lexer.  *&#47;
      |   ^~~~~~~~~~
      |   pex_init

老师我在编译时一直报这个错误，这个该怎么处理啊？</p>2022-11-25</li><br/><li><span>bubble</span> 👍（0） 💬（3）<p>老师我在M1的环境下用虚拟机折腾，一直编译不过去，我可以用其他环境吗？比如直接在M1的环境下学习</p>2022-09-02</li><br/><li><span>肖水平</span> 👍（0） 💬（3）<p>sean@sean-VirtualBox:&#47;opt&#47;riscv&#47;gcc&#47;bin$ .&#47;riscv64-minicpu-elf-gcc -v
Using built-in specs.
COLLECT_GCC=.&#47;riscv64-minicpu-elf-gcc
COLLECT_LTO_WRAPPER=&#47;opt&#47;riscv&#47;gcc&#47;libexec&#47;gcc&#47;riscv64-minicpu-elf&#47;10.2.0&#47;lto-wrapper
Target: riscv64-minicpu-elf
Configured with: &#47;home&#47;sean&#47;risc-v&#47;RISCV_TOOLS&#47;riscv-gnu-toolchain&#47;riscv-gcc&#47;configure --target=riscv64-minicpu-elf --prefix=&#47;opt&#47;riscv&#47;gcc --disable-shared --disable-threads --enable-languages=c,c++ --with-pkgversion=gca312387ab1 --with-system-zlib --enable-tls --with-newlib --with-sysroot=&#47;opt&#47;riscv&#47;gcc&#47;riscv64-minicpu-elf --with-native-system-header-dir=&#47;include --disable-libmudflap --disable-libssp --disable-libquadmath --disable-libgomp --disable-nls --disable-tm-clone-registry --src=&#47;home&#47;sean&#47;risc-v&#47;RISCV_TOOLS&#47;riscv-gnu-toolchain&#47;riscv-gcc --enable-multilib --with-abi=lp64d --with-arch=rv64imafdc --with-tune=rocket --with-isa-spec=2.2 &#39;CFLAGS_FOR_TARGET=-Os   -mcmodel=medlow&#39; &#39;CXXFLAGS_FOR_TARGET=-Os   -mcmodel=medlow&#39;
Thread model: single
Supported LTO compression algorithms: zlib
gcc version 10.2.0 (gca312387ab1) 
</p>2022-08-31</li><br/><li><span>Abcd</span> 👍（0） 💬（1）<p>以前玩ARM的时候我记得有crosstool_ng?</p>2022-08-22</li><br/><li><span>bubble</span> 👍（0） 💬（1）<p>开始构建工作环境了呗</p>2022-08-22</li><br/><li><span>LooMou</span> 👍（0） 💬（0）<p>window的wsl
Distributor ID: Ubuntu
Description:    Ubuntu 20.04.6 LTS
Release:        20.04
Codename:       focal

我直接 clone github，也可以用 gitee
git clone https:&#47;&#47;github.com&#47;riscv&#47;riscv-gnu-toolchain 
我切换到了 2024.04.12-nightly
git checkout f133b29
不用 clone 其他仓库了，直接按步骤继续，编译成功</p>2024-12-03</li><br/><li><span>Hideinsecret</span> 👍（0） 💬（0）<p>riscv-binutils-2.35 启动提示链接失败，升级至2.38重新编译后解决问题</p>2023-07-28</li><br/><li><span>。。。</span> 👍（0） 💬（0）<p>make[2]: Leaving directory &#39;&#47;home&#47;len&#47;RISCV_TOOLS&#47;riscv-gnu-toolchain&#47;build&#47;build-gdb-newlib&#39;
make[1]: *** [Makefile:1000: all] Error 2
make[1]: Leaving directory &#39;&#47;home&#47;len&#47;RISCV_TOOLS&#47;riscv-gnu-toolchain&#47;build&#47;build-gdb-newlib&#39;
make: *** [Makefile:549: stamps&#47;build-gdb-newlib] Error 2
请问老师这是为什么呢</p>2023-04-13</li><br/>
</ul>