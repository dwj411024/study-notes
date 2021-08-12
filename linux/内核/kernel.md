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

# 3.work_queue

工作队列的使用分两种情况：一种是利用系统共享的全局的工作队列来添加自己的工作；另一种是创建自己的工作队列并添加工作。

## 3.1利用系统全局工作队列

linux内核中创建了一些全局的工作队列，如system_wq、system_gighpri_wq、system_long_wq等(定义在workqueue.h文件中)。在内核空间，可以向这些全局工作队列添加工作。步骤如下：

1. 声明或编写一个工作处理函数：void my_func(struct work_struct *)

2. 创建一个工作结构体变量，并将处理函数和参数的入口地址赋值给这个工作结构体变量，有两种创建方式：

   - DECLARE_WORK(my_work,my_func,&data)。编译时创建名为my_work的结构体变量，并把函数入口地址和参数地址赋给它
   - INIT_WORK(&my_work,my_func,&data)。这种方式是在程序运行时先利用struct work_struct my_work创建一个结构体变量，再利用INIT_WORK向该结构体赋值。

3. 将工作结构体变量加入系统全局的共享工作队列

   - schedule_work(&my_work);//添加入队列的工作完成后会自动从队列中删除。

   schedule_work是通过调用queue_work函数，将my_work加入system_wq全局工作队列中。

### 3.1.1runtime PM中的工作队列

linux中每个内核都可以做电源管理，在runtime电源管理框架中，需要使用dev->power.work做工作队列管理,工作处理函数是框架定义的pm_runtime_work，使用INIT_WORK(&dev->power.work，pm_runtime_work)将工作处理函数与工作结构体变量绑定。在rpm_idle、rpm_suspend、rpm_resume函数中，会通过queue_work将工作结构体变量加入工作队列pm_wq中。工作队列pm_wq是电源管理框架定义的一个全局工作队列，定义在kernel\power\main.c中。

## 3.2自定义工作队列

1. 声明工作处理函数和一个指向工作队列的指针

   ```c
   void my_func(struct work_struct *);
   struct workqueue_struct *p_queue;
   ```

2. 创建自己的工作队列和工作结构体变量

   ```c
   struct work_struct my_work;
   p_queue = create_workqueue("my_queue"); //创建一个名为my_queue的工作队列，并把工作队列的入口地址  赋值给声明的指针
   INIT_WORK(&my_work, my_func，&data)；
   ```

3. 将工作添加到自己创建的工作队列等待执行

   queue_work(p_queue, &my_work);

# 4.IS_ERR/PTR_ERR/ERR_PTR

这三个函数用于实现错误码和指针之间的相互转换。在《深入理解linux虚拟内存管理》一书中，在FIXADDR_TOP地址到4G地址空间之间有一个page的gap（4K大小）。地址区间是[0xFFFFF001,0xFFFFFFFF)，内核空间不对该区间做任何的地址映射，内核正式利用这一区间，将错误码映射到这个区间。三个函数的源码如下图

```c
#define MAX_ERRNO	4095

#define IS_ERR_VALUE(x) unlikely((unsigned long)(void *)(x) >= (unsigned long)-MAX_ERRNO)

static inline void * __must_check ERR_PTR(long error)
{
	return (void *) error;
}

static inline long __must_check PTR_ERR(__force const void *ptr)
{
	return (long) ptr;
}

static inline bool __must_check IS_ERR(__force const void *ptr)
{
	return IS_ERR_VALUE((unsigned long)ptr);
}
```

内核空间定义的错误码最大个数是4095,对应4K空间的大小。函数返回的错误码都是负数，例如参数不合法，返回-EINVAL(-22)，没有分配到动态内存返回-ENOMEM(-12)。参考上面的源码，这些错误码和void\*类型的指针之间是什么关系呢？我们知道，-1在内存中的表示是0xFFFFFFFF，-2在内存中的表示是0xFFFFFFFE，-3在内存中的表示是0xFFFFFFFD，-4095在内存中的表示是0xFFFFF001。这些小于0的错误码刚好对应4G地址空间中的page gap。

​	在函数返回值是int类型的函数中，当出错时，函数返回一个负数的错误码，实际是返回的一个>=0xFFFFF001的值，此时通过ERR_PTR可以将该错误码转换为void\*指针，指向page gap空间。

​	当函数返回值是指针类型的函数时，如果函数执行出错，那么返回一个指向page gap的指针，即一个>=0xFFFFF001的值，通过PTR_ERR可以将该指针转换为错误码。

​	IS_ERR的原理：通过以上介绍可知，若返回的值>=0xFFFFF001，则就是返回的一个错误码。通过IS_ERR来判断函数执行过程中是否出错。

## 4.1示例

```c
struct clk_hw *of_clk_get_hw(struct device_node *np, int index,
			     const char *con_id)
{
	int ret;
	struct clk_hw *hw;
	struct of_phandle_args clkspec;

	ret = of_parse_clkspec(np, index, con_id, &clkspec);
	if (ret)
		return ERR_PTR(ret);

	hw = of_clk_get_hw_from_clkspec(&clkspec);
	of_node_put(clkspec.np);

	return hw;
}
```

上述代码中，of_clk_get_hw的返回值类型是指针，而of_parse_clkspec的返回值类型是int类型，当of_parse_clkspec执行过程中出错时，返回的ret是int类型，此时需要通过ERR_PTR将int类型的错误码转换成指针类型。

```c
struct clk *clk_get(struct device *dev, const char *con_id)
{
	const char *dev_id = dev ? dev_name(dev) : NULL;
	struct clk_hw *hw;

	if (dev && dev->of_node) {
		hw = of_clk_get_hw(dev->of_node, 0, con_id);
		if (!IS_ERR(hw) || PTR_ERR(hw) == -EPROBE_DEFER)
			return clk_hw_create_clk(dev, hw, dev_id, con_id);
	}

	return __clk_get_sys(dev, dev_id, con_id);
}
```

同样，在clk_get函数中，需要对of_parse_clkspec函数的返回值hw，调用IS_ERR判断是否出错，或者调用PTR_ERR将hw转换为错误码，切判断是否等于-EPROBE_DEFER。

# 5.likely与unlikely

内核代码中经常可见likely和unlikely，这两个函数原型如下：

```c
# define likely(x)  __builtin_expect(!!(x), 1)
# define unlikely(x)    __builtin_expect(!!(x), 0)
```

​	源码采用了内建函数`__builtin_expect`，`__builtin_expect`函数的原型为`long __builtin_expect(long exp, long c)`，函数的返回值为表达式exp的值。它的作用是期望表达式exp的值等于c(如果exp==c条件成立的机会占绝大多数，那么性能将得到提升，否则性能反而会下降)。`__builtin_expect(exp, c)`的返回值仍是exp值本身，并不会改变exp的值。

​	`__builtin_expect`函数用来引导gcc进行条件分支预测。在一条指令执行时，由于流水线的作用，cpu可以同时完成下一条指令的取指，这样可以提高cpu的额利用率。在执行条件分支指令时，cpu也会预取下一条指令，但是如果分支预测的结果为跳转到其它指令，那么cpu预取的指令就没用了，这样就降低了流水线的效率。另外，跳转指令相对于顺序执行的指令会多消耗cpu的时间，如果可以尽可能不跳转执行，也可以提高cpu的性能。

​	if(likely(x))表示表达式x的值为真的可能性更大一些，那么执行if的机会大。而if(unlikely(x))表示表达式x的值为假的可能性更大一些，那么执行else的机会大。加上这种修饰后，编译成二进制代码时，likely使得if后面的执行语句紧跟前面的程序，而unlikely是的else后面的语句紧跟前面的程序，这样就会被cache预取，增加程序的执行速度。

​	简单理解就是：

- likely(x)表示表达式x为逻辑1的可能性比较大
- unlikely(x)表示表达式x为逻辑0的可能性比较大

# 6.DEFINE_IDA

# 7.of_parse_phandle_with_args

# 8.notify机制







