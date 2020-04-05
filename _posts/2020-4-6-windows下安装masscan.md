---
layout: post
title: TCP包中究竟带了什么
---

masscan在windows下的安装

# 0x00 masscan简介
masscan是一款快速扫描器，可以6分钟扫完全部互联网
&nbsp;&nbsp;&nbsp;&nbsp;*github地址：*
&nbsp;&nbsp;&nbsp;&nbsp;[https://github.com/robertdavidgraham/masscan](https://github.com/robertdavidgraham/masscan)
但项目本身并没有提供安装包，需要自己编译，下面介绍windows上的编译
# 0x01 项目编译
使用的编译器是vs2019(别的版本同理)
*用mingW make遇到问题请直接前往0x02*
从github下载后,打开vs10文件夹中的项目
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200401112810260.png)
因为使用的是vs2019，需要自己添加编译配置
Source Files->misc->string_s.h中添加
```cpp
#if defined(_MSC_VER) && (_MSC_VER == 1925)		
//这里的_MSC_VER == 根据自己的编译器版本修改
/*Visual Studio 2019_16.5*/
# include <stdio.h>
# include <string.h>
# define strcasecmp _stricmp
# define memcasecmp _memicmp
# ifndef PRIu64
# define PRIu64 "llu"
# define PRId64 "lld"
# define PRIx64 "llx"
# endif
```
_MSC_VER是微软用来定义编译器主版本的宏定义
[https://docs.microsoft.com/en-us/cpp/preprocessor/predefined-macros](https://docs.microsoft.com/en-us/cpp/preprocessor/predefined-macros)
在这个网站中可以查到自己编译器版本对应的_MSC_VER值
修改后保存编译生成exe文件就可以正常使用
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200401115458481.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4NTQ3NzQ0,size_16,color_FFFFFF,t_70)
# 0x02 可能遇到的问题

 - 无法解析外部符号
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200401114255860.png)

**解决办法：**
这个问题是因为符号未定义造成的，本项目中缺少了一些文件未添加，手动添加即可。

以图中为例 在项目和github中分别搜索rstfilter，发现项目中只有misc-rstfilter.h没有misc-rstfilter.c导致符号无法识别
在misc文件夹右键添加现有项
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200401114934566.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4NTQ3NzQ0,size_16,color_FFFFFF,t_70)
在masscan文件夹中找到src文件夹，将以下两个文件添加进项目即可正常编译
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200401115049920.png)

 *项目文件没更新，这两个文件是后加的，文件具体功能可以自己看看_(:з」∠)_*

# 0x03 资源分享
如果你信得过我的话 也可以使用我编译好的 但**不保证能用**
[masscan](https://pan.baidu.com/s/1yp7Q4PXATBFUuejc79IJ4g) 提取码a8z1

*有问题欢迎提出*
