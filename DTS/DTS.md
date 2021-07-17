# 1.#address-cells与#size-cells

​	`#address-cells`表示用几个cells表示地址，`#size-cells`表示用几个cells表示地址长度。这两个属性主要用来描述属性`reg`中的信息。`reg`里面的个数，应该是`#address-cells + #size-cells`的整数倍。

```dtd
example 1:
/{
#address-cells=<0x1>;//在root node下使用1个u32来代表address
#size-cells=<0x0>;//在root node下使用0个u32来代表size
...
memory{
...
reg=<0x90000000>;//0x90000000是存取memory的address
...
};
};

example 2:
/{
#address-cells=<0x1>;//在root node下使用1个u32来代表address
#size-cells=<0x1>;//在root node下使用1个u32来代表size
...
memory{
...
reg=<0x90000000 0x8000>;//0x90000000是存取memory的address,0x8000是memory的size
...
};
};

example 3:
/{
#address-cells=<0x2>;//在root node下使用2个u32来代表address
#size-cells=<0x1>;//在root node下使用1个u32来代表size
...
memory{
...
reg=<0x90000000 00000000 0x8000>;//0x90000000和00000000是存取memory的address,0x8000是memory								  //的size
...
};
};
```

# 2.status属性

​	在DTS中，经常看到定义的`status`属性，当`status`属性定义为`"okay"`时，会调用相应device的probe函数，在内核中，对`status`属性的检测是通过`of/base.c`文件中的`of_device_is_available`函数实现的，在该函数中，会读取`status`属性的值，若为`"okay"`或`"ok"`，则调用相应的probe函数。

# 3.interrupt属性

## 3.1.interrupt-controller

​	这个属性为空，中断控制器应该加上此属性表明自己的身份为中断控制器。

## 3.2.interrupt-parent

​	设备结点透过该属性值来指定它所依附的中断控制器的phandle，当结点没有指定`interrupt-parent`时，则从父级结点继承。如下图，由`root`结点指定`interrupt-parent=<&intc>`，其对应于`intc`结点，而`root`结点的其它子结点一般并不会再指定`interrupt-parent`，因此他们都继承了`intc`。

```dtd
/{
    compatible = "acme,coyotes-revenge";  
    #address-cells = <1>;  
    #size-cells = <1>;  
    interrupt-parent = <&intc>;  
	...
	    intc: interrupt-controller@10140000 {  
        compatible = "arm,pl190";  
        reg = <0x10140000 0x1000 >;  
        interrupt-controller;  
        #interrupt-cells = <2>;  
    };
	...
};
```

## 3.3.#interrupt-cells

​	与`#address-cells`和`#size-cells`相似，此属性表明连接此中断控制器的设备的`interrupts`属性的cells个数

## 3.4.interrupts

​	用到了中断的设备结点通过它指定中断号、触发方式等，具体这个属性含有多少个cells，是由它依附的中断控制器的属性`#interrupt-cells`决定的。而具体每个cell又是什么含义，一般由驱动的实现决定而且也会在device tree的binding文档中说明。一般会指定中断号和中断类型等。



