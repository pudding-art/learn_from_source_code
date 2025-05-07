内核模块开发
------------

### 准备工作

-   安装开发内核环境  
    `yum install kernel-devel`，完成后根据下载的版本不同会在`/usr/src/kernels/`下生成一个类似`3.10.0-514.16.1.el7.x86_64`的目录。

-   配置编译环境  
    `` ln -sf /usr/src/kernels/3.10.0-514.16.1.el7.x86_64 /lib/modules/`uname -r`/build ``，将编译工作目录`build`正确链接。

### 最简模块开发

> 创建一个目录用于存放Makefile和其他源文件，该目录的名称与模块并没有关系。

-   编译模块的`Makefile`

```makefile
    ifneq ($(KERNELRELEASE),)
        obj-m := hellomod.o
    else
        KBUILDDIR :=/lib/modules/$(shell uname -r)/build
        PWD := $(shell pwd)
    .PHONY: module clean
    module:
        $(MAKE) -C $(KBUILDDIR) SUBDIRS=$(PWD) modules
    clean:
        $(MAKE) -C $(KBUILDDIR) SUBDIRS=$(PWD) clean
    endif
```

-   Makefile 文件解释

 1.  `KERNELRELEASE`是在内核源码的顶层Makefile中定义的一个变量，在第一次读取执行此Makefile时，`KERNELRELEASE`没有被定义，所以make将读取执行`else`之后的内容。

 2.  如果make的目标是`clean`，直接执行`clean`操作，然后结束。

 3.  当make的目标为`module`时，`-C $ (KDIR)` 指明跳转到内核源码目录下读取那里的Makefile；`M=$(PWD)` 表明然后返回到当前目录继续读入、执行当前的Makefile。当从内核源码目录返回时，`KERNELRELEASE`已被被定义，kbuild也被启动去解析kbuild语法的语句，make将继续读取`else`之前的内容。

 4.  `else`之前的内容为kbuild语法的语句,指明模块源码中各文件的依赖关系，以及要生成的目标模块名。`obj-m := hellomod.o` 表示要编译此模块的依赖对象，也是编译后模块的名称，可以直接是源文件.c对应的.o，也可以是定义的另一个依赖关系的前缀，比如在该模块有依赖多个源文件`hellomod1.c hellomod2.c`时可以是`hellomod-objs := hellomod1.o hellomod2.o`。

-   模块源代码`hellomod.c`

```c
    #include <linux/init.h>
    #include <linux/module.h>
    // 添加许可
    MODULE_LICENSE("GPL");
    // 模块作者
    MODULE_AUTHOR("ZhouKai");
    // 模块描述
    MODULE_DESCRIPTION("SimpleModule-Hello");
    static __init int hello_init(void)
    {
          printk(KERN_ALERT "Hello world\n");
          return 0;
    }
    static __exit void hello_exit(void)
    {
          printk(KERN_ALERT "Goodbye world\n");
    }
    // 模块初始化执行函数
    module_init(hello_init);
    // 模块卸载执行函数
    module_exit(hello_exit);
```

-   源代码解释  

 1.  `module_init(hello_init)`表示装载该模块的初始化函数，在执行`insmod <module_name>`时将调用该宏参数所指向的函数，返回值为整形。当返回0时，表示成功，模块也会被加载到内核空间；当返回负数时，表示出错，出错信息就是该负数绝对值对应的`errno.h`或`errno-base.h`中的宏值的错误信息，且不会被加载到内核并将被`insmod`命令打印显示该错误信息。由于内核是环境影响，该函数必须定义为`static`模式，以防止加载模块时出错。

 2.  `module_exit(hello_exit)`表示卸载该模块的清理函数，在执行`rmmod <module_name>`时调用，没有返回值。与`module_init()`一样，需要定义为`static`类型。

 3.  `MODULE_LICENSE(GPL)`模块许可证声明。如果不声明，则在模块加载时会收到内核被污染的警告，一般应遵循GPL协议。


-   编译模块  
    `make module`生成一个模块名称加`.ko`后缀的模块文件。

-   安装模块文件  

    > 就该例子而言，请通过`tailf /var/log/messages`命令查看输出信息来观察模块的装载和卸载。

 1.  `insmod hellomod.ko`将指定模块装载到内核，如果模块初始化函数返回非零，则安装失败。**如果初始化函数有严重错误，可能引起系统宕机。**

 2.  装载后将在`/sys/module/`下生成一个与该模块同名的目录，包含了一些模块信息。

-   列出已加载的模块  
    `lsmod |grep hellomod`

-   查看模块文件的信息  
    `modinfo hellomod.ko`

-   卸载模块  

 1.  `rmmod hellomod`从内核卸载指定的模块，如果没有装载则失败。

 2.  卸载后将删除`/sys/module/`下与该模块同名的目录。

-   清理编译  
    `make clean`

### 模块中参数

-   修改源文件

```c
    #include <linux/init.h>
    #include <linux/module.h>
    static char *hello_msg = "Hello word";
    // 模块参数与其描述
    module_param(hello_msg, charp, 0640);
    MODULE_PARM_DESC(hello_msg, "test module param");
    // 添加许可
    MODULE_LICENSE("GPL");
    // 模块作者
    MODULE_AUTHOR("ZhouKai");
    // 模块描述
    MODULE_DESCRIPTION("SimpleModule-Hello");
    static __init int hello_init(void)
    {
          printk(KERN_ALERT "Hello world : %s\n", hello_msg);
          return 0;
    }
    static __exit void hello_exit(void)
    {
          printk(KERN_ALERT "Goodbye world : %s\n", hello_msg);
    }
    // 模块初始化执行函数
    module_init(hello_init);
    // 模块卸载执行函数
    module_exit(hello_exit);
```

-   源代码解释

 1.  `module_param(hello_msg, charp, 0640);`定义变量的属性，该种方式会在装载后，在`/sys/module/<module_name>/parameters/`目录下生成一个与变量同名的文件，且访问权限由`module_param(var, type, mode)`的第三个参数决定。该文件的内容就是变量的值，以文本方式表示。

 2.  `MODULE_PARM_DESC(hello_msg, "test module param");`变量的文字描述，不是必须的。

-   变量的初始化  

    > 这里要介绍的初始化是通过`insmod`命令的参数灵活的指定初始化值。

 1.  正常编译

 2.  装载并指定值初始化指定的变量  
    `insmod hellomod.ko  hello_msg=2`以参数值初始化，如果不在命令行指定参数，将根据C语言的规则对命令初始化的规则进行。

 3.  通过`tailf /var/log/messages`观察结果。

-   变量关联文件系统  

    > 见源代码中的解释

-   变量参数类型列表

| name | type | comment |
|:----:|:----:|:-------:|
| `byte` | `char` | 标准类型，以字符传递参数 |
| `short` | `short` | 标准类型，以数值传递参数 |
| `ushort` | ` unsigned short` | - |
| `int` | `int` | - |
| `uint` | `unsigned int` | - |
| `long` | `long` | - |
| `ulong` | `unsigned long` | - |
| `charp` | `char *` | 以字符串传递 |
| `bool` | `bool` | 以 `0/1`、`y/n`、`Y/N` 传递，表示真假值 |

### 导出模块中的函数

-   修改hellomod模块的源码

```c
    #include <linux/init.h>
    #include <linux/module.h>

    static int hello_msg = 0;

    // 添加许可
    MODULE_LICENSE("GPL");
    // 模块作者
    MODULE_AUTHOR("ZhouKai");
    // 模块描述
    MODULE_DESCRIPTION("SimpleModule-Hello");

    void hello_say_hello()
    {
            printk(KERN_ALERT "Hello world : %s\n", hello_msg == 0 ? "default" : "user-defined");
    }
    
    static __init int hello_init(void)
    {
    //      hello_say_hello();
            printk(KERN_ALERT "Hello world : %s\n", hello_msg == 0 ? "default" : "user-defined");
            return 0;
    }
    
    static __exit void hello_exit(void)
    {
            printk(KERN_ALERT "Goodbye world : %s\n", hello_msg == 0 ? "default" : "user-defined");
    }
    
    // 模块初始化执行函数
    module_init(hello_init);
    // 模块卸载执行函数
    module_exit(hello_exit);
    
    module_param(hello_msg, int, 0640);
    MODULE_PARM_DESC(hello_msg, "test module param");
    
    EXPORT_SYMBOL(hello_say_hello);    
```

-   源代码解释  
    `EXPORT_SYMBOL(hello_say_hello)`导出一个`extern`类型的函数。

-   正常编译安装模块  

-   查看导出的符号  
    `grep hello_say_hello /proc/kallsyms`可以看见如下类似信息
    `ffffffffa022c000 T hello_say_hello      [hellomod]`
    说明已经导出符号

### 在模块中使用其他模块导出的函数

-   新增模块源码和Makefile

    > 将此模块取名为`usehellomod`
  
  -  Makefile  
  	 略

  -  源码`usehello.c`
  
```c
    #include <linux/init.h>
    #include <linux/module.h>
    #include <linux/kernel.h>
    
    
    #define CALL_EXTFUNC(func, ...) \
            { \
                    const struct kernel_symbol *_symbol = NULL; \
                    struct module *_owner = NULL; \
                    _symbol = find_symbol(#func, &_owner, NULL, true, true); \
                    if (!_symbol) return -ENOSYS; \
                    ((func)_symbol->value)(##__VA_ARGS__); \
            }
    // 添加许可
    MODULE_LICENSE("GPL");
    // 模块作者
    MODULE_AUTHOR("ZhouKai");
    // 模块描述
    MODULE_DESCRIPTION("SimpleModule-UseHello");
    
    typedef void (*hello_say_hello)();
    
    static __init int mod_init(void)
    {
            CALL_EXTFUNC(hello_say_hello);
            return 0;
    }
    
    static __exit void mod_exit(void)
    {
            printk(KERN_ALERT "Goodbye other world\n");
    }
    
    // 模块初始化执行函数
    module_init(mod_init);
    // 模块卸载执行函数
    module_exit(mod_exit);
```

-  编译装载载模块

  -  编译  
     略

  -  装载  
     在未装载`hellomod`模块时，装载此模块失败，显示与错误码`ENOSYS`一致的信息；装载`hellomod`模块后，成功装载此模块，并输出`hellomod`模块导出函数`hello_say_hello()`打印的信息。所以可以说`usehellomod`依赖`hellomod`模块。
  -  卸载  
     虽然两个模块有依赖关系，但是没有将被依赖的模块的引用增加，所以被依赖模块任然可以卸载。

### 模块依赖

通过 Makefile 来增加依赖。比如有两个驱动模块：Module A和Module B，其中Module B使用了Module A中的export的函数，因此在Module B的Makefile文件中必须添加：

```makefile
KBUILD_EXTRA_SYMBOLS += /path/to/ModuleA/Module.symvers
export KBUILD_EXTRA_SYMBOLS
```

### 使用其他模块的符号
> [参考文章](http://blog.csdn.net/figtingforlove/article/details/20150957)

-   修改`usehellomod`模块的源码

```c
    #include <linux/init.h>
    #include <linux/module.h>
    #include <linux/kernel.h>
    
    
    // 添加许可
    MODULE_LICENSE("GPL");
    // 模块作者
    MODULE_AUTHOR("ZhouKai");
    // 模块描述
    MODULE_DESCRIPTION("SimpleModule-UseHello");
    
    #define DECLARE_FUNC(func, ...) \
            typedef void (*func)(__VA_ARGS__); \
            static void *_symbol##func = NULL;
    
    
    #define CALL_FUNC(func, ...) \
    { \
            func _p = NULL; \
            if (!_symbol##func) { \
                    _symbol##func = __symbol_get(#func);\
                    if (!_symbol##func) return -ENOSYS;\
            } \
            _p = (func)_symbol##func; \
            _p (__VA_ARGS__); \
    }
    
    #define UNREF_FUNC(func) \
            do { if (_symbol##func) __symbol_put(#func); } while(0)
    
    DECLARE_FUNC(hello_say_hello, void);
    
    static __init int mod_init(void)
    {
            CALL_FUNC(hello_say_hello);
            return 0;
    }
    
    static __exit void mod_exit(void)
    {
            UNREF_FUNC(hello_say_hello);
            printk(KERN_ALERT "Goodbye other world\n");
    }
    
    // 模块初始化执行函数
    module_init(mod_init);
    // 模块卸载执行函数
    module_exit(mod_exit);
```

-   源码解释
  1.  在装载初始化函数中，通过使用`__symbol_get()`获取符号的地址（这里为导出函数地址），并将其保存在本地静态变量中（此种静态变量不需要与文件系统关联，但是能在`/proc/kallsyms`看到）。执行次函数成功后被查找符号所在模块的引用增加一次，可以通过`lsmod hellomod`查看验证，输出信息如下`hellomod 12737 1`，此时使用`rmmod hellomod`将失败，报告`rmmod: ERROR: Module hellomod is in use`这样的错误信息。

  2.  在卸载退出函数中，通过使用`__symbol_put()`（传递`__symbol_get()`获取符号的地址作为参数）可以相应减少一次引用，如果引用为0，则可以成功卸载模块。