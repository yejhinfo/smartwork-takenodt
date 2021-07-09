

# lcd驱动模型分析

### 1. APP和内核隔离

rt-smart中，APP无法直接访问硬件寄存器，必须通过驱动程序来操作硬件。
而APP和内核是隔离的，APP无法直接调用内核中驱动程序的函数。
APP使用驱动的流程如下图所示：

![001_rtt_app_driver](D:\workspace\takenodt\lcd\pic\001_rtt_app_driver.png)

在smart系统中，app是无法直接访问硬件的，而是通过系统提供的一套io接口来对硬件设备进行操作的（是在应用程序看来是这样）。通过一整套标准的接口来操作硬件。

## 2.对lcd的驱动分析

第一，先对lcd的ops结构体进行分析，可以看出，lcd驱动只实现了，两个对硬件的操作接口。因此在应用层调用的时候，只有两个接口可以使用

```c
1.rt_device_init();
2.rt_device_control();
```

```c
/*实现的驱动接口*/
1.static rt_err_t imx6ull_elcd_init(rt_device_t device)
2.static rt_err_t imx6ull_elcd_control(rt_device_t device, int cmd, void *args)
```

![02](D:\workspace\takenodt\lcd\pic\02.png)

## 3.来分析imx6ull_elcd_init（）

这个函数主要是来配置相应的lcd的硬件参数。

1.如配置lcd的时钟，利用 **CLOCK_InitVideoPll(&pll_config);**来配置lcd所需的时钟频率

2.利用 **ELCDIF_RgbModeInit(elcd_dev->config->ELCDIF, &lcd_config);ELCDIF_RgbModeStart(elcd_dev->config->ELCDIF);**；来配置相应的lcd参数，比如屏幕的宽，高，bbp等

![03](D:\workspace\takenodt\lcd\pic\03.png)

### 4.分析int rt_hw_elcd_init(void)

![05](D:\workspace\takenodt\lcd\pic\05.jpg)

​                       

​                          ![09](D:\workspace\takenodt\lcd\pic\09.jpg)   
​                           所以这个宏定义的值#define PV_OFFSET 0xc0000000 是0xc0000000

​                                                                     ![04](D:\workspace\takenodt\lcd\pic\04.png)

![06](D:\workspace\takenodt\lcd\pic\06.png)



### 5.imx6ull_elcd_control

`rt_hw_kernel_phys_to_virt`这个是之前的接口，已经不用了

frambuffer是内核态的地址，此时用户态不能直接访问的，如果要访问该怎么办？此时应用了一个共享内存页面的特性，将一块内核态的内存与用户态的内存进行共享，此时访问就可以正常进行了。

![08](D:\workspace\takenodt\lcd\pic\08.png)

