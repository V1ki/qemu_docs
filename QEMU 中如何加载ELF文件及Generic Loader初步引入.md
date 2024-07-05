
# QEMU 中如何加载ELF文件及Generic Loader初步引入.

# 写在最前
这一章的构造的时间比较久, 最开始我只是想写下`ELF`文件的读取,但是感觉这样的话,和前面的博客的内容的过渡太生硬了. 同时不知道大家在看到之前的博客中的命令时,会不会好奇这些命令是怎么执行的, 我这边是比较好奇的 ,同时为了更加了解这部分的内容, 所以稍微了解了下.

# QEMU Command Option 追踪流程

在之前的博客中, `.vscode/tasks.json` 中通过配置了`QEMU`的命令来调试:
```bash
<Path>/build/qemu-system-arm -machine mps2-an385 -cpu cortex-m3 -kernel ${workspaceFolder}/build/gcc/output/RTOSDemo.out -monitor none -nographic -serial stdio -s -S
```
那这些命令,`QEMU`中是怎么解析处理的呢, 这里以` -kernel`以及` -device` 进行举例.

## QEMU 命令配置 

根目录下的`qemu-option.hx`中定义了 `device` 以及`kernel` 的用法.      
`kernel` 选项的用法 : 
```
SRST

The kernel options were designed to work with Linux kernels although other things (like hypervisors) can be packaged up as a kernel executable image. The exact format of a executable image is usually architecture specific.

The way in which the kernel is started (what address it is loaded at, what if any information is passed to it via CPU registers, the state of the hardware when it is started, and so on) is also architecture specific. Typically it follows the specification laid down by the Linux kernel for how kernels for that architecture must be started.

ERST

DEF("kernel", HAS_ARG, QEMU_OPTION_kernel, \
    "-kernel bzImage use 'bzImage' as kernel image\n", QEMU_ARCH_ALL)
SRST
``-kernel bzImage``
    Use bzImage as kernel image. The kernel can be either a Linux kernel
    or in multiboot format.
ERST

```

`device`选项的描述比较多, 这里只截取一部分 :
```

DEF("device", HAS_ARG, QEMU_OPTION_device,
    "-device driver[,prop[=value][,...]]\n"
    "                add device (based on driver)\n"
    "                prop=value,... sets driver properties\n"
    "                use '-device help' to print all possible drivers\n"
    "                use '-device driver,help' to print all possible properties\n",
    QEMU_ARCH_ALL)
SRST
``-device driver[,prop[=value][,...]]``
    Add device driver. prop=value sets driver properties. Valid
    properties depend on the driver. To get help on possible drivers and
    properties, use ``-device help`` and ``-device driver,help``.

    Some drivers are:
....
ERST
```
> 其他的选项也是类似, 找到对应的`DEF("xxxx"....)`即可. 

在`meson.build`中可以看到, 在编译时,通过`hxtool`将`qemu-options.hx`转换成`qemu-options.def`: 
```
hxdep = []
hx_headers = [
  ['qemu-options.hx', 'qemu-options.def'],
  ['qemu-img-cmds.hx', 'qemu-img-cmds.h'],
]
if have_system
  hx_headers += [
    ['hmp-commands.hx', 'hmp-commands.h'],
    ['hmp-commands-info.hx', 'hmp-commands-info.h'],
  ]
endif
foreach d : hx_headers
  hxdep += custom_target(d[1],
                input: files(d[0]),
                output: d[1],
                capture: true,
                command: [hxtool, '-h', '@INPUT0@'])
endforeach
genh += hxdep
```
而转换工具`hxtool`则是`hxtool = find_program('scripts/hxtool')`, 继续查看`scripts/hxtool`中的代码:
```/bin/sh
#!/bin/sh

hxtoh()
{
    flag=1
    while read -r str; do
        case $str in
            HXCOMM*)
            ;;
            SRST*|ERST*) flag=$(($flag^1))
            ;;
            *)
            test $flag -eq 1 && printf "%s\n" "$str"
            ;;
        esac
    done
}

case "$1" in
"-h") hxtoh ;;
*) exit 1 ;;
esac < "$2"

exit 0
```

编译后, 可以将`qemu-options.hx`转换成`qemu-options.def`, 并且在`system/vl.c` 中通过`include`的方式引入. 

可以看到在`system/vl.c`中看到枚举相关的代码:
```c

enum {

#define DEF(option, opt_arg, opt_enum, opt_help, arch_mask)     \
    opt_enum,
#define DEFHEADING(text)
#define ARCHHEADING(text, arch_mask)

#include "qemu-options.def"
};
```
所以,在后面的代码中,我们只需要关注`opt_enum`, 在这篇博客中,暂时只关注`QEMU_OPTION_kernel`和`QEMU_OPTION_device`.

## QEMU 命令处理

在`system/vl.c`中的`qemu_init`方法中,观察`switch(popt->index)`对应的`case` ,这个就是上面看到的`opt_enum`即`QEMU_OPTION_kernel`等.

这里先说`kernel`的配置, 首先找到`QEMU_OPTION_kernel`的case, 可以看到,其对应的代码非常简单:
```c
qdict_put_str(machine_opts_dict, "kernel", optarg);
```
即,将`kernel`对应的参数放置到`machine_opts_dict`中, 而随后, 通过`qemu_create_machine`创建对应的`machine`, 由于`Machine`实际也是遵守`QOM`的规则, 所以这里可以找到对应的`Machine`类型:`mps2-an385`,`TYPE_MPS2_AN385_MACHINE` , 它的`parent`为`TYPE_MPS2_MACHINE`, 再上一层为`TYPE_MACHINE`, 然后我们在`hw/core/machine.c`的`machine_class_init`中可以看到:
```c
object_class_property_add_str(oc, "kernel", machine_get_kernel, machine_set_kernel);
object_class_property_set_description(oc, "kernel", "Linux kernel image file");
```
这里给对应的`class`增加了名为`kernel`的`StringProperty`, 其`get`和`set`方法分别对`machine_get_kernel`及`machine_set_kernel`:`
```c

static char *machine_get_kernel(Object *obj, Error **errp)
{
    MachineState *ms = MACHINE(obj);

    return g_strdup(ms->kernel_filename);
}

static void machine_set_kernel(Object *obj, const char *value, Error **errp)
{
    MachineState *ms = MACHINE(obj);

    g_free(ms->kernel_filename);
    ms->kernel_filename = g_strdup(value);
}
```

而后,在`qemu_apply_machine_options`中,通过`object_set_properties_from_keyval`这个方法,可以循环调用`machine_opts_dict`对应的`class`中的属性的`set`方法, 至此, 我们就已经将命令行中的`kernel`的参数设置到了`MachineState` 中的`kernel_filename`字段中了. 而后, 则是在`mps2_common_init`中,通过`armv7m_load_kernel`将`elf`进行读取并处理.


使用方式: `-device loader,file=<elf file path>,cpi-num=<cpu-num>`

参考文档: `docs\system\generic-loader.rst`


# ELF 简诉

## 为什么要了解ELF 文件格式
在学习和使用`QEMU` , 通常会加载`kernel` 文件,  但是具体是怎么加载的? 主要加载什么内容 ? QEMU 是根据什么来加载镜像的 . 

## 什么是ELF 
ELF ( Executable and Linkable Format/Extensible Linking Format )  定义了二进制文件，库和核心文件。 形式化规范允许操作系统正确地解释其底层机器指令。ELF文件通常是编译器或链接器的输出，并且是二进制格式。 


##  ELF 文件结构
首先使用`file` 查看文件类型:
```bash
$ file test.elf
test.elf: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), statically linked, with debug_info, not stripped
```
使用`readelf` 来看查看文件结构, 这里只简单的提下`ELF Header`, 可以使用`readelf -h` 读取`ELF Header` : 
```bash
$ readelf -h .\build\gcc\output\RTOSDemo.out
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           ARM
  Version:                           0x1
  Entry point address:               0x2719
  Start of program headers:          52 (bytes into file)
  Start of section headers:          345116 (bytes into file)
  Flags:                             0x5000200, Version5 EABI, soft-float ABI
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         2
  Size of section headers:           40 (bytes)
  Number of section headers:         227
```
- 前4个字节以固定的头`7F 45 4C 46`开始 .  
- 第5个字节 就是判断文件的Architecture, 即 `32位`或者`64位` . 其中`1`表示`32位` , `2`表示`64位`. 
- 第6个字节为 数据字段, 用于判断 大小端. `1`表示LSB (小端), `2`表示`MSB`(大端).
- 第7个字节为 固定值`1` ,版本,  目前只支持`1`.




# 参考资料
- https://en.wikipedia.org/wiki/Executable_and_Linkable_Format
- https://linux-audit.com/elf-binaries-on-linux-understanding-and-analysis/
- http://www.skyfree.org/linux/references/ELF_Format.pdf
- https://qemu-project.gitlab.io/qemu/devel/qom.html