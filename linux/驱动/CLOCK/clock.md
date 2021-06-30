# 1.常用辅助函数

CCF框架在clk-provider.h文件中提供了一些辅助函数，可以方便开发者的使用，常用的有以下几个：

* const char *__clk_get_name(const struct clk *clk);由clk的指针获取该时钟的名字。
* const char *clk_hw_get_name(const struct clk_hw *hw);由hw指针获取该时钟的名字。
* struct clk_hw *__clk_get_hw(struct clk *clk);由clk指针获取指向clk_hw的指针。
* unsigned int clk_hw_get_num_parents(const struct clk_hw *hw);由hw指针获取该时钟的父时钟的个数
* struct clk_hw *clk_hw_get_parent(const struct clk_hw *hw);由hw指针获取该时钟的父时钟的hw指针
* struct clk_hw *clk_hw_get_parent_by_index(const struct clk_hw *hw, unsigned int index);当hw指针指向的时钟有多个父时钟时，通过index指定想要获取的父时钟编号。
* struct clk *__clk_lookup(const char *name);由时钟的名字获取该时钟的clk指针



