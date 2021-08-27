# 1.交叉编译工具

不同的平台需要不同的交叉编译工具链，但是工具链的组成基本是相同的，都包含gcc、gdb、ld、objcopy、objdump等工具，只是不同的工具链其前缀不同罢了。常见的工具及其作用如下(以arm平台的编译工具`arm-linux-`为例进行说明)：

- gcc：编译的前端程序，它通过调用其它程序来实现将程序源文件编译成目标文件。编译时，它首先调用预处理程序`arm-linux-cpp`对输入的源程序进行处理，然后再将预处理后的程序编译成汇编代码，最后由`arm-linux-as`将汇编代码编译成目标代码。
- gdb：调试工具，具有非常丰富使用调试命令，非常好用
- ld：链接工具
- cpp：预处理工具
- as：将汇编语言转换成可重定位目标代码
- ar:库管理器。将多个可重定位的目标模块归档为一个函数库文件，在linux中，就是用来生成静态库.o文件或动态库.so文件的。
- objcopy:用来拷贝一个目标文件的内容到另一个文件中，可以使用不同于源文件的格式来输出目的文件，即可以进行格式转换。
- objdump:用于显示二进制文件信息，常用来查看反汇编代码。

# 2.arm-linux-gcc与arm-elf-gcc

gcc(The GNU Compiler Collection)，是一套由GNU开发的编译器集，为什么是编译器集而不是编译器呢？因为它不仅支持C语言编译，还支持c++，Ada，Objective C等语言。gcc内部主要由Binutils、gcc-core、Glibc等软件包组成。

- Binutils：它是一组开发工具，包括链接器、汇编器和其它用于目标文件和档案的工具。该软件包依赖于不同的目标机平台，因为不同目标机的指令集是不一样的。
- gcc-core:该部分只包含c的编译器及公共部分，而对其它语言的支持包需要另外安装。gcc-core依赖于Binutils。
- Glibc:包含了主要的c库

例如一个简单的hello world程序。预处理和编译主要由gcc-core来完成，汇编和链接主要有Binutils完成。那么何时用到glibc呢？源码中的printf打印函数，这个函数在gcc中是以库函数的形式存在的，这个库函数在glibc中。

那么arm-linux-gcc和arm-elf-gcc什么区别呢？二者的主要区别是在于使用不同的c库文件。arm-linux-gcc使用GNU的glibc，而arm-elf-gcc是使用uClibc。因而arm-linux-* 和 arm-elf-*区别主要表现在C语言库的实现上，例如不同系统调用，不同的函数集实现，不同的ABI/启动代码以及不同系统特性等微小的差别。

# 3.configure文件的build/host/target

- build:执行代码编译的主机，一般是自己的pc机或编译服务器。这个参数一般由config.guess来猜就可以，当然也可以自己指定。
- host:编译出来的二进制程序所执行的主机，如果是本机编译，本机执行，那这个值就等于build。只有交叉编译的时候，才会build和host不同。用host指定运行主机。
- target:这个选项只有在建立交叉编译环境的时候用到，正常编译和交叉编译都不会用到。它是用build主机上的编译器，便宜一个新的编译器，这个新的编译器将来编译出来的其他程序将运行在target指定的系统上。

至于这三个参数的值，一般是交叉编译器的公共部分的前缀，例如，riscv的交叉编译工具的前缀都是riscv64-unknown-linux-gnu*，所以对应的这三个参数的值就是riscv64-unknown-linux。

## 3.1示例

以编译gcc为例进行说明

`./configure --build=i386-linux --host=powerpc-linux --target=powerpc-linux`

说明：利用i386-linux(--build)的编译器对gcc进行编译，编译出来的gcc运行在powerpc-linux(--host)，这个gcc用来编译能够在powerpc-linux(--target)上运行的代码

`./configure --build=i386-linux --host-i386-linux --target=powerpc-linux`

说明：利用i386-linux(--build)的编译器对gcc进行编译，编译出来的gcc运行在i386-linux(--host)，这个gcc用来编译能够在powerpc-linux(--target)上运行的代码



