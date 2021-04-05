# 1.snd_ctl_open

原型：`static int snd_ctl_open(struct inode *inode, struct file *file)`当打开control设备时，就会调用该函数。①该函数通过`snd_lookup_minor_data`函数获取snd_card类型的声卡的指针。②调用`snd_card_file_add`函数，该函数将打开的文件加入snd_card的files_list链表，用于跟踪连接状态，避免热插拔释放繁忙的资源。③申请一个struct snd_ctl_file类型的结构体，用于保存snd_card等信息，并把该结构体的指针赋值给file->private_data。④将该snd_ctl_file加入card->ctl_files链表中。

# 3.snd_ctl_release

与`snd_ctl_open`函数作用相反，释放各个control资源。

> 注意：该函数中的control->count表示当前的control实例中有几个element。

# 4.file、snd_ctl_file及card->controls的关系

# 5.snd_ctl_new与snd_ctl_new1

两个函数都是创建一个control实例，不同的是：`snd_ctl_new`是通过传递的参数进行初始化，而`snd_ctl_new1`是通过传递的一个模板进行初始化。`snd_ctl_new`会申请一个snd_kcontrol类型的结构体，然后将access和file参数赋值给新申请的结构体。`snd_ctl_new1`函数中会调用`snd_ctl_new`，除此之外，还会将模板中的各个变量赋值给