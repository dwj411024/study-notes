# 1.pr_fmt(fmt)

内核中很多驱动都在第一行定义pr_fmt(fmt)。它的主要作用是在输出的log输出一些额外的固定信息。在使用dev_dbg输出log时，输出的log都会有device的name，pr_fmt的作用与此相同。例如，在i2c的驱动文件i2c-core-base.c中，定义`#define pr_fmt(fmt) "i2c-core: " fmt`。在使用`pr_emerg`、`pr_alert`、`pr_err`、`pr_info`等进行打印时，会自动在打印信息前加上"i2c-core: "信息。示例如下：

```c
#define pr_fmt(fmt) "gpio_demo: " fmt

static int gpio_demo_probe(struct platform_device *pdev) 
{
    ...
    pr_info("%s enter.\n", __func__);
}
```

打印结果为：`gpio_demo: gpio_demo_probe enter.`

# 2.container_of

## 2.1.作用

内核中经常会遇到的一个宏container_of(ptr, type, member)。该宏定义的作用是：已知结构体type的成员member的地址ptr，求解结构体type的起始地址。

​		type的起始地址 = ptr - size (此处都需要转换为char*)

container_of的原型中涉及到0指针的使用。&((type *)0)->member的作用就是求member成员到结构体type起始地址的字节数。可以这么理解：(type *)0是将0强制转化为指向type类型的指针，即在0地址处定义了一个type类型的结构体，那么，&((type *)0)->member得到的是member成员的地址，又因为该结构体起始地址为0，所以即为member成员到type结构体起始地址的字节数。

## 2.2.内核编程的严谨

```c
#define container_of(ptr, type, member) ({
const typeof(((type *)0)->member) *__mptr = (ptr);
....
})
```

上面第2行的作用是什么？为什么要定义一个__mptr呢？如果开发者使用时输入的参数有问题：ptr与member的类型不匹配，编译时就会有warnning，但是如果去掉该行，就没有warnning了，而这个警告恰恰是必须的。typeof(((type *)0)->member)的作用仅仅是获取member的类型而已。

# 3.DEFINE_IDA

# 4.of_parse_phandle_with_args

# 5.notify机制







