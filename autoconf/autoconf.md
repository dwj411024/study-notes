# 1.编译出错

使用autogen.sh编译时，总是报undefined AC_XXXXX，出现这种问题，首先检查autoconf的各个依赖软件是否已安装，主要检查以下几项，并确保安装的软件不低于以下版本号：

|  autoconf  |  2.68  |
| :--------: | :----: |
|  automake  |  1.11  |
|  libtool   | 1.5.22 |
|     m4     | 1.4.16 |
| pkg-config |  0.25  |

