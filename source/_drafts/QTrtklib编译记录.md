---
title: QTrtklib编译记录
tags:
- rtklib 
- Qt
---

> 结合最近编译中遇到的问题，重新试一下看能不能编译通过rtklib的Qt版本。毕竟用c++build编译太麻烦了。
>
> 需要特别说明的是本文使用的代码是rtklibexplorer修改的rtklib代码分支，所以有些问题可能是直接编译rtklib源码所不能遇到的。
<!--more-->
使用的代码：

* rtklib-[rtklibexplorer分支](https://github.com/rtklibexplorer/RTKLIB)（以下简称exp分支）
* rtklib-[主分支](https://github.com/tomojitakasu/RTKLIB)

## 问题1：

如下图：

![问题1](https://raw.githubusercontent.com/guoxianwei/guoxianwei.github.io/picGo/pictures/qt问题1.png))

* 方案1： 从之前的版本中找到这个文件，放到rcv下，然后修改`src.pro`，注释掉`rcv/ss2.c \`z这行代码再反注释就可以初步调试通。

* 方案2：删除`rcv/ss2.c \`这行代码，即认为不存在这个文件进行编译。
  
## 问题2：

  找不到FREQ1的定义。


思路：查看该问题位于ss2.c文件中。（也就是上步注销掉ss2.c文件就没这个问题了。。）由于ss2.c文件很久没有更新了，所以追溯到最后一次更新的版本，找到对应的`rtklib.h`
[头文件](https://github.com/tomojitakasu/RTKLIB/blob/b416a94bd2fdf8b1d82124c0b43b81b1ae8e4031/src/rtklib.h)71行。

 ```c
  #define FREQ1       1.57542E9           /* L1/E1  frequency (Hz) *
 ```

对应现在的rtklib.h文件对应的变量为`FREQL1`:

  ```c
  #define FREQL1      1.57542E9           /* L1/E1  frequency 1 (Hz) */
  ```

  所以解决办法为：将`FREQ1`修改为`FREQL1`.


## 问题3：

code2obs调用方法不对： 

![](https://raw.githubusercontent.com/guoxianwei/guoxianwei.github.io/picGo/pictures/qt问题3.png)

思路：这个是因为exp分支对北斗的频率做了特殊处理：

```c
extern char *code2obs(int sys, unsigned char code, int *freq)
{
    if (freq) *freq=0;
    if (code<=CODE_NONE||MAXCODE<code) return "";
    if (freq) {
       if (sys==SYS_CMP)
           *freq=obsfreqs_cmp[code];
       else
           *freq=obsfreqs[code];
    }
    return obscodes[code];
}
```

主分支为：

```c
extern char *code2obs(unsigned char code, int *freq)
{
    if (freq) *freq=0;
    if (code<=CODE_NONE||MAXCODE<code) return "";
    if (freq) *freq=obsfreqs[code];
    return obscodes[code];
}
```

特殊处理在添加了北斗对应的观测频段`obsfreqs_cmp`数组。从上面函数的变化可以看出，新增加的参数主要是针对北斗的观测值，所以只有当使用北斗观测值的时候才对`sys`赋值`SYS-CMP`，例如： 

```c
obs1=code2obs(SYS_CMP,codes[i],NULL);
```

除北斗外的观测值sys赋值为0即可。例如：

```c
obs1=code2obs(0,codes[i],NULL);
```

所以找到这些报错的地方，根据函数使用环境，在sys变量对应位置填上`0`或者`SYS_CMP`即可。(由于这里只是在调用数据绘图，BDS和Galileo的频段很多类似，所以统一补充上0就行)。

## 问题4：

> RTKLIB-demo5\app\rtkconv_qt\codeopt.cpp:333: error: 'FREQTYPE_L7' was not declared in this scope
>      E27->setEnabled((NavSys&SYS_GAL)&&(FreqType&FREQTYPE_L7));

查看主分支的源代码里的`codeopt.cpp`这一部分也没改，检索发现`rtklib.h`解释的`FREQTYPE_L7`定义为：  

```c
#define FREQTYPE_L7 0x10                /* frequency type: E5b/B2 */
#define FREQTYPE_L8 0x20                /* frequency type: E5(a+b) */
```

而exp分支对它做了处理，所以有点不一样，特殊处理了一下： 

```c
#define FREQTYPE_E5b 0x04               /* frequency type: E5b/B2 */
```

按照同样的方法处理其他提示频率有问题的项目： 

```c
#define FREQTYPE_L6 0x08                /* frequency type: E6/LEX/B3 */
#define FREQTYPE_E6 0x10                /* frequency type: E6/LEX/B3 */
```

```c
#define FREQTYPE_L8 0x20                /* frequency type: E5(a+b) */
#define FREQTYPE_E5ab 0x20              /* frequency type: E5(a+b) */
```

```c
#define FREQTYPE_L9 0x40                /* frequency type: S */
#define FREQTYPE_S 0x40                 /* frequency type: S */
```

## 问题5 :

> error: No rule to make target '../../src/debug/libRTKLib.a', needed by 'release/rtkget_qt.exe'.  Stop.

在网上查找的时候遇到过国外的一个小伙伴遇到这个问题，帖子暂时找不到了，不过找到一个国内的小伙伴调试的帖子这个可能有用：[Windows环境下的RTKPlot_Qt版本编译时遇到的问题和解决方法](http://www.fdlly.com/p/1800078394.html),这里就参照这个帖子一步一步调试看看： 



就是有一点需要注意：`libRTKLib.lib`生成需要调换到`debug`模式，然后在`构建`菜单下执行`qmake`操作。

debug模式下调试没问题，在*****——debug目录下的app/debug/rtk各软件目录下能找到各自编好的debug版本的程序，但是直接运行不行，会提示缺库，这是qt打包exe时会出现的问题，参考这个[帖子](https://www.cnblogs.com/ourran/p/6524790.html)，这时候需要用到用到这个工具: ![qt](https://raw.githubusercontent.com/guoxianwei/guoxianwei.github.io/picGo/pictures/Snipaste_2019-06-09_20-43-09.png)

在命令行下执行`windeployqt xxx.exe`,xxx这里可以放上生成的exe dug版的路径。执行完后这个exe文件夹内会生成多个dll库，这时候双击exe就可以运行了。这就类似生成的是绿色版的程序。对，类似Vnote的Windows版，就是直接运行exe。

确实感觉没有c++ builder 生成的方便。另外，切换到release模式，一直会提示找不到`libRtkLib.a`，实在搞不定就放弃了。

看到科学网的这篇博客，突然发现一个点：mkl矩阵加速，链接是：[RTKLIB 在OSX系统下Xcode/CLion平台上的编译问题及解决方法](http://wap.sciencenet.cn/home.php?mod=space&uid=2958868&do=blog&id=1076413),好奇查了一下好像确实能[加快矩阵运算](https://blog.csdn.net/verystory/article/details/76660460?utm_source=blogxgwz8),但因为mkl是Intel的技术，商用还是有风险。

