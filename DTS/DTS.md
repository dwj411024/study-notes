# 1.#address-cells与#size-cells

`#address-cells`表示用几个cells表示地址，`#size-cells`表示用几个cells表示地址长度。这两个属性主要用来描述属性`reg`中的信息。`reg`里面的个数，应该是`#address-cells + #size-cells`的整数倍。

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
}
}

example 2:
/{
#address-cells=<0x1>;//在root node下使用1个u32来代表address
#size-cells=<0x1>;//在root node下使用1个u32来代表size
...
memory{
...
reg=<0x90000000 0x8000>;//0x90000000是存取memory的address,0x8000是memory的size
...
}
}

example 3:
/{
#address-cells=<0x2>;//在root node下使用2个u32来代表address
#size-cells=<0x1>;//在root node下使用1个u32来代表size
...
memory{
...
reg=<0x90000000 00000000 0x8000>;//0x90000000和00000000是存取memory的address,0x8000是memory								  //的size
...
}
}
```

