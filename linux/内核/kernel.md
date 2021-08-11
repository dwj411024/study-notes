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

# 5.DEFINE_IDA

# 6.of_parse_phandle_with_args

# 7.notify机制







