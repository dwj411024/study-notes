# 1.snd_minors

在/core/sound.c文件中，定义了一个全局的指针数组`static struct snd_minor *snd_minors[SNDRV_OS_MINORS]`。该数组的值会在`snd_register_device`函数中进行初始化，即每有一个device进行初始化，就会有一个snd_minors指针指向该device。而`snd_register_device`在control设备和pcm设备注册时都会调用该函数，即每一个control设备和pcm设备都有一个snd_minors中的元素与其对应。

> 注意：在注册control设备时，由snd_ctl_dev_register调用snd_register_device传递的private_data是snd_card类型，所以在snd_ctl_open函数中，通过snd_lookup_minor_data查找的SNDRV_DEVICE_TYPE_CONTROL类型的private_data是snd_card类型的。而在注册pcm设备时，由snd_pcm_dev_register调用snd_register_device传递的private_data是snd_pcm类型的，在pcm_native.c文件中，通过调用snd_lookup_minor_data查找snd_pcm类型的参数。

# 2.iminor函数

该函数的作用是通过struct inode结点返回设备的设备号。在struct inode结点中，成员变量i_rdev的含义是：如果inode代表设备，则i_rdev表示该设备的设备号。iminor函数就是返回inode结点的i_rdev的值。

# 3.一些基本概念

- frame：音频中一帧为一个采样点的数据量，通常以byte为单位表示。

  立体声48KHZ，16bit的PCM流一帧数据是4byte：2（立体声）*16bit/8=4byte

  5.1声道48KHZ，16bit的PCM流一帧数据是12byte：6（5.1声道）*16bit/8=12bytes

- period：每两次硬件中断间的帧的个数。poll()每个period返回一次。

- 音频的buffer是一个环形buffer。通常设置为2*period大小。例如，硬件被设置为48KHZ，2个period，每个period有1024帧，那么buffer一般需要设置到2048帧。硬件在一个buffer大小内会产生两次中断。

- buffer_size = period_size * periods

  period_bytes = period_size * bytes_per_frame

  bytes_per_frame = channels * bytes_per_sample

  periods表示buffer_size中有几个period，而上面说的period（每两次中断间的帧的个数）其实就是此处的period_size。periods定义了更新状态的频率，通常是通过中断触发状态的更新。period_size定义了一个period内的帧的个数。在音频硬件中，环形缓冲区会被分成几部分，并在每部分的边界触发irq中断。period_size定义了这个块的大小。

> 5.1声道：指六声道环绕声。包括中央声道，前置左、右声道，后置左、右声道，以及0.1声道重低音声道。一套系统可接6个喇叭，一般用于各类影院中。

该部分内容可参考ALSA官网：alsa-project.org/wiki/FramesPeriods