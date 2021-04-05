# 1.snd_pcm_devices链表

同control设备类似，在pcm.c中定义了一个snd_pcm_devices链表，所有注册的pcm设备都会挂载到该链表上。函数`snd_pcm_get`在该链表上查找指定设备号的pcm;`snd_pcm_next`通过查找该链表，返回指定设备号的下一个pcm设备的设备号;`snd_pcm_add`通过该链表将新的pcm添加到链表中，如果链表中该设备号已被占用，则该新pcm不能挂载。

> snd_pcm_devices链表中，挂载的pcm设备是按照设备号的大小由小到大挂载的，所以在snd_pcm_next函数中，查找到pcm->device > device，就认为找到了下一个设备。在snd_pcm_add函数中，添加新的pcm实例，是通过查找设备号，如果找到的设备号比新的pcm实例的设备号大，那么就把新的pcm设备添加到查找到的设备号前。

# 2.snd_pcm_new_stream

原型：`int snd_pcm_new_stream(struct snd_pcm *pcm, int stream, int substream_count)`。

**pcm**:pcm实例

**stream**:stream的方向

**substream_count**:substream的个数

创建一个新的pcm stream及其下的substream。调用该函数前，pcm实例的相应的流必须是空的，也就是说，`snd_pcm_new`的参数必须是空的。此函数是用来创建substream。①该函数根据stream的方向，先将pcm实例的streams的地址赋值给snd_pcm_str类型的pstr,然后在对pstr所指向的snd_pcm_str结构体进行初始化。②然后调用`snd_device_initialize`初始化struct device结构体。③调用`dev_set_name`设置pcm实例中该流的名字**pcmCxDxp**或**pcmCxDxc**。

> 一个pcm实例对应两个pcm stream。两个pcm stream分别有一个名字(**pcmCxDxp**或**pcmCxDxc**),相应的对应两个设备。

④根据substream_count的值，在for循环中依次申请substream空间，对substream的值进行初始化，并把substream加入到链表中。

# 3.snd_pcm_new

原型：`int snd_pcm_new(struct snd_card *card, const char *id, int device,
		int playback_count, int capture_count, struct snd_pcm **rpcm)`。该函数只是简单的调用`_snd_pcm_new`，具体的实现在`_snd_pcm_new`函数。在`_snd_pcm_new`函数中：①定义snd_device_ops结构体的操作函数，依据是否是创建内部pcm实例而有所不同，`snd_pcm_new`创建的是非内部的pcm实例。②申请pcm实例的空间；③对pcm实例的结构体进行初始化，赋值设备号，pcm所属的声卡，是否为内部pcm等。④根据playback_count和capture_count的值，两次调用`snd_pcm_new_stream`，对pcm的stream进行赋值及申请substream空间，创建substream。⑤调用`snd_device_new`，创建snd_device类型的设备组件，该设备组件主要用于保存device_data和ops等信息。然后由后向前遍历声卡snd_card的devices链表。将snd_device插入到合适位置。

# 4.snd_pcm_dev_register

原型：`static int snd_pcm_dev_register(struct snd_device *device)`。函数的执行流程是：①获取snd_pcm指针，因为在调用`snd_device_new`时，是将指向snd_pcm类型的指针pcm作为device_data的，并将pcm赋值给snd_device类型的device_data成员。所以，在`snd_pcm_dev_register`函数中，device->device_data就是snd_pcm类型的指针。②调用`snd_pcm_add`将该pcm添加到链表中。③在for循环中，两次调用`snd_register_device`,分别注册playback和capture

> 注意：在调用snd_register_device时，传递的ops是snd_pcm_f_ops，该数组定义在pcm_native.c文件中，传递的private_data是pcm，该private_data会赋值给snd_minor类型的成员变量private_data，所以，通过全局变量snd_minors数组，就可以得到相应的pcm指针。