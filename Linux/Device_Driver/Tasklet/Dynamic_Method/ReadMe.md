This is just a basic linux device driver. This will explain tasklet dynamic method in the linux device driver.

Please refer this URL for the complete tutorial of this source code.
https://embetronicx.com/tutorials/linux/device-drivers/tasklets-dynamic-method/


Building and Testing Driver
Build the driver by using Makefile (sudo make)
Load the driver using sudo insmod driver.ko
To trigger the interrupt read device file (sudo cat /dev/etx_device)
Now see the Dmesg (dmesg)
linux@embetronicx-VirtualBox: dmesg
[12372.451624] Major = 246 Minor = 0
[12372.456927] Device Driver Insert...Done!!!
[12375.112089] Device File Opened...!!!
[12375.112109] Read function
[12375.112134] Shared IRQ: Interrupt Occurred
[12375.112139] Executing Tasklet Function : arg = 0
[12375.112147] Device File Closed...!!!
[12377.954952] Device Driver Remove...Done!!!
We can able to see the print “Shared IRQ: Interrupt Occurred“ and “Executing Tasklet Function: arg = 0“
Unload the module using sudo rmmod driver



使用内联汇编将中断 11 引发到内核模块时出错
https://stackoverflow.com/questions/57391628/error-while-raising-interrupt-11-with-inline-asm-into-kernel-module
这曾经适用于较旧的内核版本，但在更高版本上失败。原因是通用 IRQ 处理程序 do_IRQ() 已更改，以获得更好的 IRQ 处理性能。它不是使用 irq_to_desc() 函数来获取 IRQ 描述符，而是从每个 CPU 的数据中读取它。描述符在物理设备初始化期间放置在那里。由于这个伪设备驱动程序没有物理设备，因此 do_IRQ() 在那里找不到它并返回错误。如果我们想使用软件中断来模拟IRQ，我们必须首先将IRQ描述符写入每个CPU的数据中。不幸的是，符号 vector_irq（每个 CPU 数据中的 IRQ 描述符数组）在内核编译期间不会导出到内核模块。改变它的唯一方法是重新编译整个内核。如果您认为值得付出努力，可以添加以下行：

EXPORT_SYMBOL (vector_irq);
在文件中：arch/x86/kernel/irq.c

就在所有包含行之后。编译并从新编译的内核引导后，按如下方式更改驱动程序：

添加包含行：

    #include <asm/hw_irq.h>
将读取函数更改为：

static ssize_t etx_read(struct file *filp,
                char __user *buf, size_t len, loff_t *off)
{
        struct irq_desc *desc;

        printk(KERN_INFO "Read function\n");
        desc = irq_to_desc(11);
        if (!desc) return -EINVAL;
        __this_cpu_write(vector_irq[59], desc);
        asm("int $0x3B");  // Corresponding to irq 11
        return 0;
}


