原始字符设备驱动
================

   1. 手动创建设备节点

   2. 手动分配设备号

代码构成
--------

寄存器虚拟映射
~~~~~~~~~~~~~~

.. code:: c

   static void __iomem* V_GPIO1_DR;
   V_GPIO1_DR = ioremap(PHY_ADDR);//映射
   iounmap(V_GPIO1_DR)  //退出模块时要取消映射

读写寄存器
~~~~~~~~~~

读写寄存器数据，操作的是经过\ ``ioremap``\ 映射的地址。有一组读/写寄存器的函数，包含对8位/16位/32位等寄存器的操作。以下是\ ``32``\ 位寄存器的操作方法。

.. code:: c

   readl(address)
   writel(val,address)//address是虚拟地址
   iowrite32()  
   ioread32()

定义设备操作函数
~~~~~~~~~~~~~~~~

实现设备操作函数

.. code:: c

   static int led_open(struct inode *inode, struct file *filp);
   static int led_close(struct inode *inode, struct file *filp);
   static ssize_t led_read(struct file *filp, char __user *buf,size_t cnt, loff_t *offset) ;
   static ssize_t led_write(struct file *filp, const char __user *buf, size_t cnt, loff_t *offset);

然后定义一个设备操作集合，并把函数指针赋值给对应成员变量

.. code:: c

   static struct file_operations led_ops{
       .owner = THIS_MODULE,
       .open = led_open,
       .release = led_close,
       .write = led_write,
       .read = led_read,
   };

用户空间和内核空间传递数据
~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: c

   retvalue =copy_from_user(databuf, buf, cnt);

实现注册设备驱动
~~~~~~~~~~~~~~~~

.. code:: c

   static int __init led_init(void)
   {
   	//初始化外设寄存器
   	//注册驱动
   	retvalue = register_chrdev(LED_MAJOR, NAME,&led_ops);
   }

实现退出设备驱动
~~~~~~~~~~~~~~~~

先实现退出设备驱动的函数

.. code:: c

   static void __exit led_exit(void)
   {
   	iounmap(V_GPIO1_GDIR);//取消虚拟地址的映射
   	unregister_chrdev(LED_MAJOR, NAME); //取消注册字符驱动
   }

然后声明驱动入口

.. code:: c

   module_init(led_init);
   module_exit(led_exit);
   MODULE_AUTHOR("lorenzo");
   MODULE_LICENSE("GPL");

新字符设备驱动
==============

   1. 自动创建设备节点

   2. 自动分配设备号

涉及的几个结构体
----------------

   1. file_operations --->定义了对设备的操作函数

   2. cdev --->和初始化字符设备相关

   3. class ---->和自动创建设备节点相关

   4. device --->和自动创建设备节点相关

驱动模块入口涉及的函数
----------------------

注册字符设备
~~~~~~~~~~~~

1. 自己指定了设备号\ ``led_device.devid = MKDEV(led_device.major,0);  //组合主设备号次设备号``

   .. code:: c

      register_chrdev_region(led_device.devid, LED_CNT, LED_NAME);

2. 自动分配设备号

   .. code:: c

      alloc_chrdev_region(&led_device.devid, 0, LED_CNT, LED_NAME);

初始化字符设备
~~~~~~~~~~~~~~

.. code:: c

   led_device.cdev.owner = THIS_MODULE;
   cdev_init(&led_device.cdev, &led_ops);//初始化字符设备
   cdev_add(&led_device.cdev, led_device.devid, LED_CNT);//向系统添加字符设备

以上\ **注册字符设备**\ 和\ **初始化字符设备**\ 就是实现的\ ``register_chrdev``\ 函数的功能。

自动创建节点
~~~~~~~~~~~~

1. 创建类

   .. code:: c

      led_device.class = class_create(THIS_MODULE, LED_NAME);
      if(IS_ERR(led_device.class)) {
              return PTR_ERR(led_device.class);
      }

2. 创建设备节点

   .. code:: c

      led_device.device = device_create(led_device.class, NULL, 
                                        led_device.devid, NULL, LED_NAME);
      if(IS_ERR(led_device.device)) {
          return PTR_ERR(led_device.device);
      }

驱动模块出口涉及的函数
----------------------

**省略**\ 了和以前旧的字符设备驱动相同的部分(取消虚拟地址映射)。

1. 删除字符设备

2. 取消字符设备的注册

3. 摧毁设备

4. 摧毁类

.. code:: c

   cdev_del(&led_device.cdev);
   unregister_chrdev_region(led_device.devid, LED_CNT);

   device_destroy(led_device.class, led_device.devid);
   class_destroy(led_device.class);

文件私有数据
------------

.. code:: c

   /* open 函数 */
   static int test_open(struct inode *inode, struct file *filp)
   {
       filp->private_data = &testdev; /* 设置私有数据 */
       return 0;
   }

在 open 函数里面设置好私有数据以后，在 write、 read、 close
等函数中直接读取 private_data即可得到设备结构体。

带设备树的字符驱动
==================

增加设备树信息
--------------

针对某一款芯片具体的设备数文件在\ ``arch/arm/boot/dts/``\ 。

在\ ``imx6ull-alientek-emmc.dts``\ 对根节点进行追加节点：

.. code:: c

   alphaled {
   		#address-cells = <1>;
   		#size-cells = <1>;
   		compatible = "atkalpha-led";
   		status = "okay";
   		reg = <	0X020C406C 0X04 /* CCM_CCGR1_BASE */
   				0X020E0068 0X04 /* SW_MUX_GPIO1_IO03_BASE */
   				0X020E02F4 0X04 /* SW_PAD_GPIO1_IO03_BASE */
   				0X0209C000 0X04 /* GPIO1_DR_BASE */
   				0X0209C004 0X04 >; /* GPIO1_GDIR_BASE */
   	};

其中\ ``reg``\ 的就是要操作的外设的具体寄存器。
比如\ ``0X020C406C``\ 表示寄存器地址，\ ``0X04``\ 表示该寄存器的长度为4字节。主要是为了进行虚拟地址映射时方便。

在驱动入口函数获取设备树信息
----------------------------

linux
kernel提供了一组对特定类型的属性进行操作的函数对设备树进行操作，定义在\ ``linux/of.h``\ 里面。

主要涉及的两个结构体为
``struct device_node``\ 和\ ``struct property``\ 。

.. code:: c

   led.node = of_find_node_by_path("/alphaled"); //寻找节点
   if(led.node == NULL) {
       printk("alphaled is not found!\r\n");
   }

   proper = of_find_property(led.node, "compatible", NULL);//寻找节点的属性
   if(proper == NULL) {
       printk("compatible property is not found!\r\n");
   }

   ret = of_property_read_string(led.node, "status", &str);//读取字符串类型的属性
   if(ret < 0 ) {
       printk("failed to read status!\r\n");
   }
   //读取u32类型的数组
   ret = of_property_read_u32_array(led.node, "reg", regdata, 10);
   if(ret < 0) {
       printk("failed to read reg!\r\n");
   }

虚拟化地址
~~~~~~~~~~

使用\ ``void __iomem *of_iomap(struct device_node *device, int index)``\ 直接对节点的reg属性进行虚拟地址转换。

.. code:: c

   V_CCM_CCGR1 = of_iomap(led.node,0);//转换reg属性第一个寄存器
   V_SW_MUX_GPIO1_IO3 = of_iomap(led.node,1);//转换reg属性第二个寄存器
   V_SW_PAD_GPIO1_IO03 = of_iomap(led.node,2);
   V_GPIO1_DR = of_iomap(led.node,3);
   V_GPIO1_GDIR = of_iomap(led.node,4);

PINCTRL和GPIO子系统
===================

PINCTRL子系统
-------------

pinctrl用来配置gpio的复用功能、上下拉、速度、驱动能力等属性。我们只需在设备树文件中添加或修改某个IO的配置就可以，怎么对IO进行操作是由内核的PINCTRL模块完成的，pinctrl源码在\ ``drivers/pinctrl``\ 。

首先在iomux节点下添加要配置的引脚：

.. code:: 

     /* led pinctrl by lorenzo */
   		pinctrl_led: ledgrp {
   			fsl,pins = <
   				MX6UL_PAD_GPIO1_IO03__GPIO1_IO03 0x10B0
   			>;
   		};

其中\ ``MX6UL_PAD_GPIO1_IO03__GPIO1_IO03``\ 定义在\ ``imx6ul-pinfunc.h``\ 里，这是按指定格式写好的寄存器信息，格式为\ ``<mux_reg conf_reg input_reg mux_mode input_val>``\ ，含义分别给为：mux寄存器地址、conf寄存器地址、input寄存器地址、要写入mux寄存器的值、要输入input寄存器的值。上面的\ ``0x10B0``\ 即要写入conf寄存器的值。

.. code:: c

   #define MX6UL_PAD_GPIO1_IO03__GPIO1_IO03          0x0068 0x02F4 0x0000 0x5 0x0

**这只是针对imx6ull这款处理器是这样，估计不同处理器的配置方法不一样，具体需要参考内核源码里针对具体芯片的pinctrl绑定文档。例如imx6ul关于pinctrl的绑定文档为\ ``Documentation/devicetree/bindings/pinctrl/fsl,imx6ul.txt``\ 。**\ 里面有针对imx6ull的pinctrl配置的详细信息如下：

   Required properties for pin configuration node:

   -  fsl,pins: each entry consists of 6 integers and represents the mux
      and config setting for one pin. The first 5 integers **<mux_reg
      conf_reg input_reg mux_val input_val>** are specified using a
      PIN_FUNC_ID macro, which can be found in imx*-pinfunc.h under
      device tree source folder. The last integer CONFIG is the pad
      setting value like pull-up on this pin. And that's why fsl,pins
      entry looks like <PIN_FUNC_ID CONFIG> in the example below.

GPIO子系统
----------

``PINCTRL``\ 子系统只要配置IO的复用功能、上下拉、速度、驱动能力等属性。\ ``PINCTRL``\ 将IO复用为GPIO了之后，就需要GPIO子系统来配置了。

在根节点下面添加需要用到的GPIO。

.. code:: c

   gpioled {
   	#address-cells = <1>;
   	#size-cells = <1>;
   	compatible = "atkalpha-gpioled";  
   	pinctrl-names = "default";
   	pinctrl-0 = <&pinctrl_led>; //这里引用了pinctrl的设置
   	led-gpio = <&gpio1 3 GPIO_ACTIVE_HIGH>; //这是指定gpioled使用的引脚编号
   	status = "okay"; //表示设备是可操作的
   };

然后进行设备树编译,并将设备树加载到开发板，接下来就可以通过\ **简化**\ 的函数接口来操作gpio了。

驱动编写
--------

驱动结构并无多大改变，主要替换了一些函数接口。

1. 定义了一个\ ``int``\ 类型的的GPIO号。

   .. code:: 

      struct gpioled
      {
          dev_t devid;
          int major;
          int minor;
          struct cdev cdev;
          struct class *class;
          struct device *device;
          struct device_node *node;
          
          int led_gpio; /* the gpio index of the led */

      };

2. 设备入口函数可以获取设备树中定义的gpio号，并进行初始化操作

   .. code:: c

      static int __init led_init (void) {

          int ret = 0;

          gpioled.node = of_find_node_by_path("/gpioled");
          if(gpioled.node == NULL) {
              printk("gpioled node can not found!\r\n");
          }

          gpioled.led_gpio = of_get_named_gpio(gpioled.node, "led-gpio", 0); //获取设备树中定义的GPIO的编号，至于led_gpio等于多少不需要知道
          if(gpioled.led_gpio < 0) {
              printk("can not get gpio-led");
              return -EINVAL;
          }
          printk("led-gpio num is %d\r\n",gpioled.led_gpio);

          ret = gpio_direction_output(gpioled.led_gpio, 1);  //设置gpio的方向
          if(ret < 0) {
              printk("can not set gpio to high!\r\n");
          }
      	//以下是注册设备的一系列操作，和以前程序一样
          if(gpioled.major) {
              gpioled.devid = MKDEV(gpioled.devid, 0);
              register_chrdev_region(gpioled.devid, 1, GPIOLED_NAME);
          } else {
              alloc_chrdev_region(&gpioled.devid, 0, GPIOLED_CNT, GPIOLED_NAME);
          }
              .........
                  
          return 0;
          }

3. 使用简化的函数接口代替了写寄存器操作

   .. code:: c

      static ssize_t led_write (struct file *filp, const char __user *buf, size_t cnt, loff_t *off) {

          unsigned char data;
          
          struct gpioled *dev = filp->private_data;

          int err = copy_from_user(&data, buf, cnt);
          if(err < 0 ) {
              printk("kernel write failed!\r\n");
              return -EFAULT;
          }

          if(data == LEDON) {
              gpio_set_value(dev->led_gpio, 0);  //通过该函数设置GPIO为低电平以打开LED
          } else if (data == LEDOFF) {
              gpio_set_value(dev->led_gpio, 1);
          }

          return 0;
      }

并发
====

原子操作
--------

原子操作就是指不能在进一步分割的操作
，用来操作\ **整形数据和位**\ 操作。用来保证同一时刻只有一个\ **线程或者任务**\ 操作该变量。

1. 对整形的操作

   +----------------------------------+----------------------------------+
   | 函数                             | 描述                             |
   +==================================+==================================+
   | ATOMIC_INIT(int i)               | 定义原子变量的时候对其初始化。   |
   +----------------------------------+----------------------------------+
   | int atomic_read(atomic_t \*v)    | 读取 v 的值，并且返回。          |
   +----------------------------------+----------------------------------+
   | void atomic_set(atomic_t \*v,    | 向 v 写入 i 值。                 |
   | int i)                           |                                  |
   +----------------------------------+----------------------------------+
   | void atomic_add(int i, atomic_t  | 给 v 加上 i 值。                 |
   | \*v)                             |                                  |
   +----------------------------------+----------------------------------+
   | void atomic_sub(int i, atomic_t  | 从 v 减去 i 值。                 |
   | \*v)                             |                                  |
   +----------------------------------+----------------------------------+
   | void atomic_inc(atomic_t \*v)    | 给 v 加 1，也就是自增。          |
   +----------------------------------+----------------------------------+
   | void atomic_dec(atomic_t \*v)    | 从 v 减 1，也就是自减            |
   +----------------------------------+----------------------------------+
   | int atomic_dec_return(atomic_t   | 从 v 减 1，并且返回 v 的值。     |
   | \*v)                             |                                  |
   +----------------------------------+----------------------------------+
   | int atomic_inc_return(atomic_t   | 给 v 加 1，并且返回 v 的值。     |
   | \*v)                             |                                  |
   +----------------------------------+----------------------------------+
   | int atomic_sub_and_test(int i,   | 从 v 减 i，如果结果为 0          |
   | atomic_t \*v)                    | 就返回真，否则返回假             |
   +----------------------------------+----------------------------------+
   | int atomic_dec_and_test(atomic_t | 从 v 减 1，如果结果为 0          |
   | \*v)                             | 就返回真，否则返回假             |
   +----------------------------------+----------------------------------+
   | int atomic_inc_and_test(atomic_t | 给 v 加 1，如果结果为 0          |
   | \*v)                             | 就返回真，否则返回假             |
   +----------------------------------+----------------------------------+
   | int atomic_add_negative(int i,   | 给 v 加                          |
   | atomic_t \*v)                    | i，                              |
   |                                  | 如果结果为负就返回真，否则返回假 |
   +----------------------------------+----------------------------------+

2. 对位的操作

   +----------------------------------+----------------------------------+
   | 函数                             | 描述                             |
   +==================================+==================================+
   | void set_bit(int nr, void \*p)   | 将 p 地址的第 nr 位置 1。        |
   +----------------------------------+----------------------------------+
   | void clear_bit(int nr,void \*p)  | 将 p 地址的第 nr 位清零。        |
   +----------------------------------+----------------------------------+
   | void change_bit(int nr, void     | 将 p 地址的第 nr 位进行翻转。    |
   | \*p)                             |                                  |
   +----------------------------------+----------------------------------+
   | int test_bit(int nr, void \*p)   | 获取 p 地址的第 nr 位的值。      |
   +----------------------------------+----------------------------------+
   | int test_and_set_bit(int nr,     | 将 p 地址的第 nr 位置            |
   | void \*p)                        | 1，并且返回 nr 位原来的值。      |
   +----------------------------------+----------------------------------+
   | int test_and_clear_bit(int nr,   | 将 p 地址的第 nr                 |
   | void \*p)                        | 位清零，并且返回 nr 位原来的值。 |
   +----------------------------------+----------------------------------+
   | int test_and_change_bit(int nr,  | 将 p 地址的第 nr                 |
   | void \*p)                        | 位翻转，并且返回 nr 位原来的值。 |
   +----------------------------------+----------------------------------+

自旋锁(spin lock)
-----------------

.. _特点-1:

特点
~~~~

1. 没获取到锁的线程会\ **持续等待**\ (**阻塞**)，降低系统性能。一般在\ **持有锁时间不长的场景**\ 下使用。

2. 所保护的临界区内\ **不能引起该线程休眠或阻塞**\ ，否则将会使其他线程无法获得锁，造成\ **死锁。**

3. 在\ **获取锁之前需要关闭中断**\ ，否则在执行临界区操作时有中断发生，而且该中断也获取同一个锁，将导致死锁发生。

4. **不能递归申请自旋锁**\ ，这样会导致自己把自己锁死。

5. 不管用的是单核或多核的SOC，都将其当做多核SOC来编写驱动程序。

6. 中断中只能使用自旋锁。

7. 一般在线程中使用\ ``spin_lock_irqsave/spin_unlock_irqrestore``\ ，在中断中使用
   ``spin_lock/spin_unlock``\ 。

.. _api-1:

API
~~~

+----------------------------------+----------------------------------+
| 函数                             | 描述                             |
+==================================+==================================+
| DEFINE_SPINLOCK(spinlock_t lock) | 定义并初始化一个自选变量。       |
+----------------------------------+----------------------------------+
| int spin_lock_init(spinlock_t    | 初始化自旋锁。                   |
| \*lock)                          |                                  |
+----------------------------------+----------------------------------+
| void spin_lock(spinlock_t        | 获取指定的自旋锁，也叫做加锁。   |
| \*lock)                          |                                  |
+----------------------------------+----------------------------------+
| void spin_unlock(spinlock_t      | 释放指定的自旋锁。               |
| \*lock)                          |                                  |
+----------------------------------+----------------------------------+
| int spin_trylock(spinlock_t      | 尝试获取指                       |
| \*lock)                          | 定的自旋锁，如果没有获取到就返回 |
|                                  | 0                                |
+----------------------------------+----------------------------------+
| int spin_is_locked(spinlock_t    | 检查指定的自                     |
| \*lock)                          | 旋锁是否被获取，如果没有被获取就 |
|                                  | 返回非 0，否则返回 0。           |
+----------------------------------+----------------------------------+

和中断相关的函数：

+----------------------------------+----------------------------------+
| 函数                             | 描述                             |
+==================================+==================================+
| void spin_lock_irq(spinlock_t    | 禁止本地中断，并获取自旋锁。     |
| \*lock)                          |                                  |
+----------------------------------+----------------------------------+
| void spin_unlock_irq(spinlock_t  | 激活本地中断，并释放自旋锁。     |
| \*lock)                          |                                  |
+----------------------------------+----------------------------------+
| void                             | 保存中断状                       |
| spin_lock_irqsave(spinlock_t     | 态，禁止本地中断，并获取自旋锁。 |
| \*lock, unsigned long flags)     |                                  |
+----------------------------------+----------------------------------+
| void                             | 将中断状态恢复                   |
| s                                | 到以前的状态，并且激活本地中断， |
| pin_unlock_irqrestore(spinlock_t | 释放自旋锁                       |
| \*lock, unsigned long flags)     |                                  |
+----------------------------------+----------------------------------+

使用示例
~~~~~~~~

.. code:: c

   DEFINE_SPINLOCK(lock) /* 定义并初始化一个锁 */
      
      /* 线程 A */
   void functionA (){
       unsigned long flags; /* 中断状态 */
       spin_lock_irqsave(&lock, flags) /* 获取锁 */
       /* 临界区 */
       spin_unlock_irqrestore(&lock, flags) /* 释放锁 */
   }
      
       /* 中断服务函数 */
   void irq() {
       spin_lock(&lock) /* 获取锁 */
       /* 临界区 */
       spin_unlock(&lock) /* 释放锁 */
   }

其他锁
------

   都是自旋锁的衍生，所以\ **具有自旋锁的特点**\ 。

读写锁
~~~~~~

1. 每次只能一个读操作或者写操作，但是可以并发读取的。

顺序锁
~~~~~~

1. 允许写的时候进行读，但是不运行两个线程同时写。

2. 在读的过程中发生了写操作，最好重新进行读取，保证数据完整性。

3. 顺序锁保护的资源不能是指针 。

信号量(semaphore)
-----------------

信号量是一种同步机制。一般的信号量指计数信号量，二值信号量实际上就是值为1的计数信号量。

.. _特点-2:

特点
~~~~

1. 信号量可以使线程进入休眠。

2. 由于信号量使线程进入休眠，因此会有线程切换，所以开销比自旋锁大。共享资源持有时间比较短时，尽量不使用信号量。

3. 信号量会引起休眠，所以\ **不能在中断中**\ 使用。

4. 临界区中可以调用引起阻塞的API。

.. _api-2:

API
~~~

+----------------------------------+----------------------------------+
| 函数                             | 描述                             |
+==================================+==================================+
| DEFINE_SEAMPHORE(name)           | 定义                             |
|                                  | 一个信号量，并且设置信号量的值为 |
|                                  | 1。                              |
+----------------------------------+----------------------------------+
| void sema_init(struct semaphore  | 初始化信号量 sem，设置信号量值为 |
| \*sem, int val)                  | val。                            |
+----------------------------------+----------------------------------+
| void down(struct semaphore       | 获取信号量，因为会               |
| \*sem)                           | 导致休眠，因此不能在中断中使用。 |
+----------------------------------+----------------------------------+
| int down_trylock(struct          | 尝试获取信号量，如               |
| semaphore \*sem);                | 果能获取到信号量就获取，并且返回 |
|                                  | 0。如果不能就返回非 0，并且      |
|                                  | 不会进入休眠。                   |
+----------------------------------+----------------------------------+
| int down_interruptible(struct    | 获取信号量，和 down              |
| semaphore \*sem)                 | 类似，只是使用 down 进           |
|                                  | 入休眠状                         |
|                                  | 态的线程不能被信号打断。而使用此 |
|                                  | 函数                             |
|                                  | 进入休眠以后是可以被信号打断的。 |
+----------------------------------+----------------------------------+
| void up(struct semaphore \*sem)  | 释放信号量                       |
+----------------------------------+----------------------------------+

.. _示例-1:

示例
~~~~

.. code:: c

   struct semaphore sem; /* 定义信号量 */
   sema_init(&sem, 1)； /* 初始化信号量 */
   down(&sem); /* 申请信号量 */
   /* 临界区 */
   up(&sem); /* 释放信号量 */

互斥体(Mutex)
-------------

互斥体又叫互斥信号量，和二值信号量具有相同功能。

.. _特点-3:

特点
~~~~

1. 能引起休眠，因此不能在中断中使用。

2. 临界区中可以调用引起阻塞的API。

3. 必须由信号持有者释放，否则导致死锁。

4. 不能递归持有或释放。

.. _api-3:

API
~~~

+----------------------------------+----------------------------------+
| 函数                             | 描述                             |
+==================================+==================================+
| DEFINE_MUTEX(name)               | 定义并初始化一个 mutex 变量。    |
+----------------------------------+----------------------------------+
| void mutex_init(mutex \*lock)    | 初始化 mutex。                   |
+----------------------------------+----------------------------------+
| void mutex_lock(struct mutex     | 获取 mutex，也就是给 mutex       |
| \*lock)                          | 上锁。如果获 取不到就进休眠。    |
+----------------------------------+----------------------------------+
| void mutex_unlock(struct mutex   | 释放 mutex，也就给 mutex 解锁。  |
| \*lock)                          |                                  |
+----------------------------------+----------------------------------+
| int mutex_trylock(struct mutex   | 尝试获取 mutex，如果成功就返回   |
| \*lock)                          | 1，如果失 败就返回 0。           |
+----------------------------------+----------------------------------+
| int mutex_is_locked(struct mutex | 判断 mutex                       |
| \*lock)                          | 是否被获取，如果是的话就返回     |
|                                  | 1，否则返回 0。                  |
+----------------------------------+----------------------------------+
| int                              | 使用此                           |
| mutex_lock_interruptible(struct  | 函数获取信号量失败进入休眠以后可 |
| mutex \*lock)                    | 以被信号打断                     |
+----------------------------------+----------------------------------+

.. _示例-2:

示例
~~~~

.. code:: c

   struct mutex lock; /* 定义一个互斥体 */
   mutex_init(&lock); /* 初始化互斥体 */

   mutex_lock(&lock); /* 上锁 */
   /* 临界区 */
   mutex_unlock(&lock); /* 解锁 */

下载程序的操作
==============

TFTP获取宿主机文件
------------------

Linux
~~~~~

下载单个文件到开发板：tftp -g -r filename IP //IP为window IP

上传单个文件到pc端：tftp -p -l filename IP //IP为window IP

.. code:: shell

   tftp -g -r led.ko 192.168.0.105

uboot
~~~~~

uboot下使用tftp服务需要提前设置好板子的网络信息：

.. code:: shell

   setenv ipaddr 192.168.1.106
   setenv ethaddr 00:04:9f:04:d2:35
   setenv gatewayip 192.168.1.1
   setenv netmask 255.255.255.0
   setenv serverip 192.168.1.105
   saveenv  

然后才能通过tftp下载宿主机上的文件

.. code:: shell

   tftp 80800000 zImage

更新镜像
--------

在uboot中更新内核
~~~~~~~~~~~~~~~~~

.. code:: shell

   tftp 80800000 zImage
   fatwrite mmc 1:1 80800000 zImage 0x5c2720 //写入EMMC
   fatls mmc 1:2

在uboot中更新设备树
~~~~~~~~~~~~~~~~~~~

.. code:: shell

   tftp 83000000 imx6ull-alientek-emmc.dtb

从网络启动
~~~~~~~~~~

启动之前必须设置\ ``bootargs``\ 环境变量，否则会找不到根文件系统而提示\ ``kernel panic``\ 。

.. code:: shell

   setenv bootargs 'console=ttymxc0,115200 root=/dev/mmcblk1p2 rootwait rw'
   setenv bootcmd 'tftp 80800000 zImage; tftp 83000000 imx6ull-alientek-emmc.dtb; bootz 80800000 - 83000000'
   saveenv

通过nfs挂载根文件系统
~~~~~~~~~~~~~~~~~~~~~

.. code:: c

   setenv bootargs 'console=ttymxc0,115200 root=/dev/nfs rw nfsroot=192.168.0.105:/home/lorenzo/imx_learning/nfs/rootfs ip=192.168.0.104:192.168.0.105:192.168.0.1:255.255.255.0::eth0:off'

格式为：

.. code:: 

   root=/dev/nfs nfsroot=[<server-ip>:]<root-dir>[,<nfs-options>] ip=<client-ip>:<server-ip>:<gwip>:<netmask>:<hostname>:<device>:<autoconf>:<dns0-ip>:<dns1-ip>

驱动模块安装和移除
------------------

1. 安装

   .. code:: shell

      insmod xxx.ko
      mknod /dev/xxx c 200 0  # 设备名字 设备类型 主设备号 次设备号

   ``mknod``\ 针对没有自动创建设备节点的驱动。自动创建设备节点的驱动不需要\ ``mknod``\ 。

2. 卸载

   .. code:: shell

      rmnod xxx.ko
