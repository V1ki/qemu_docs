# 写在最前
这几天一直在研究`QEMU`中多核ARM加载不同镜像的问题, 一直不得其解, 这部分后续可以分几个不分拆解下,看看为什么会出现这种问题. 今天先来看看如何使用`QEMU`来调试`FreeRTOS`的示例代码.


# 编译并运行 FreeRTOS 示例代码 (基础版本)

首先是下载代码, 这种只需要看最新代码情况下,我通常会使用`--depth=1`来减少下载的代码量; 并且需要下载子模块, 所以使用`--recurse-submodules`来下载子模块.
```shell 
git clone https://github.com/FreeRTOS/FreeRTOS.git --recurse-submodules --depth=1
```
然后进入到 `FreeRTOS/Demo/CORTEX_MPS2_QEMU_IAR_GCC`中, 在这个目录下,使用`vscode`打开. 我比较喜欢使用`MSYS2 MinGW 64-bit` 控制台, 但是`vscode`中`bash (MSYS2)`打开后,默认不是在当前工作区的目录, 所以可以通过修改`Preferences: Open User Settings (JSON)`的方式打开用户设置`settings.json`, 然后找到`"terminal.integrated.profiles.windows"`这个配置, 然后添加如下配置:
```json
  "bash mingw64": {
      "path": "C:\\msys64\\msys2_shell.cmd",
      "args": [
          "-defterm",
          "-here",
          "-no-start",
          "-mingw64"    
      ]
  },
```

由于这个目录下已经配置好了`launch.json`和`tasks.json`, 但是这里需要简单的调整一下, 让其在我们编译出的`QEMU`中运行, 打开`tasks.json`文件:
```json
{
    // See https://go.microsoft.com/fwlink/?LinkId=733558
    // for the documentation about the tasks.json format
    "version": "2.0.0",
    "tasks": [
        {
          "label": "Build QEMU",
          "type": "shell",
          "command": "make --directory=${workspaceFolder}/build/gcc",
          "problemMatcher": {
            "base": "$gcc",
            "fileLocation": ["relative", "${workspaceFolder}/build/gcc"]
          },
          "group": {
              "kind": "build",
              "isDefault": true
          }
        },
        {
          "label": "Run QEMU",
          "type": "shell",
          "command": "echo 'QEMU RTOSdemo started'; qemu-system-arm -machine mps2-an385 -cpu cortex-m3 -kernel ${workspaceFolder}/build/gcc/output/RTOSDemo.out -monitor none -nographic -serial stdio -s -S",
          "dependsOn": ["Build QEMU"],
          "isBackground": true,
          "problemMatcher": [
            {
              "pattern": [
                {
                  "regexp": ".",
                  "file": 1,
                  "location": 2,
                  "message": 3
                }
              ],
              "background": {
                "activeOnStart": true,
                "beginsPattern": ".",
                "endsPattern": "QEMU RTOSdemo started",
              }
            }
          ]
        }
    ]
}
```
我们需要将`qemu-system-arm`调整为我们编译出的`QEMU`的路径:
```json
"command": "echo 'QEMU RTOSdemo started'; <Path>/build/qemu-system-arm -machine mps2-an385 -cpu cortex-m3 -kernel ${workspaceFolder}/build/gcc/output/RTOSDemo.out -monitor none -nographic -serial stdio -s -S"
```
到这里, 我们就可以通过`F5`来运行我们的`FreeRTOS`示例代码了, 由于`launch.json`中配置了`preLaunchTask`为`Run QEMU`, 而`Run QEMU`又依赖于`Build QEMU`, 所以在运行时会先编译, 然后运行`QEMU`, 这里的参数更加复杂:
```shell
 <Path>/build/qemu-system-arm -machine mps2-an385 -cpu cortex-m3 -kernel ${workspaceFolder}/build/gcc/output/RTOSDemo.out -monitor none -nographic -serial stdio -s -S
```
在`基础版`中,我们只关注运行, 但是也需要简单的调试, 所以设置 `-monitor none` , `-nographic` , `-serial stdio` , `-s` , `-S` 这几个参数, 这里简单的解释一下:
- `-monitor none` : 不显示监视器
- `-nographic` : 不显示图形界面
- `-serial stdio` : 将串口输出重定向到标准输出
- `-s` : 启动GDB服务器
- `-S` : 启动时暂停

我一般不喜欢贴图, 主要是因为贴图不方便修改, 但是这里还是需要简单的贴一下:
![Debug Simple](./images/使用%20QEMU%20调试%20FreeRTOS示例/1.png)


其实我在代码中,并未加入任何的断点, 但是启动后, 仍然停在了`Reset_Handler`中, 这是因为在`launch.json`中将`stopAtEntry`设置为了`true`, 这个参数的意思是在程序启动时, 是否停在入口处, 这里我们设置为`true`, 所以会停在`Reset_Handler`中, 如果设置为`false`, 则会直接运行程序, 这里大家可以自行测试.

关于断点相关的调试, 这里就不多赘述了, 至于GDB的内容, 后续需要的时候,再来详细的介绍.

# 进阶调试版本 (初次接触Monitor)

在这一个篇章中,我们可以初步了解下`QEMU Monitor` , `QEMU Monitor`是一个用于与`QEMU`交互的命令行工具, 现阶段我们主要通过`QEMU Monitor`查看`QEMU`的状态.

为此, 我们需要调整之前在`tasks.json` 中`Run QEMU`的命令, 这里有几种选择:
1. 使用独立的`QEMU Monitor`窗口

为此, 首先,我们需要将`-nographic`去掉, 这样`QEMU`就会显示图形界面, 然后其次,我们要将`-monitor none`去掉, 改成`-monitor vc` , 改动后`command`如下:
```json
"command": "echo 'QEMU RTOSdemo started'; <Path>/build/qemu-system-arm -machine mps2-an385 -cpu cortex-m3 -kernel ${workspaceFolder}/build/gcc/output/RTOSDemo.out -monitor vc -serial stdio -s -S"
```
当然,这种效果和直接去掉`-nographic`以及`-monitor none`是一样的, 但是为了明确我们使用了`QEMU Monitor`, 我们这里使用了`-monitor vc`.

按照这种方式, 启动的`QEMU Monitor`窗口,不要关闭, 因为如果关闭, `QEMU`也会关闭, 那么调试就结束了.

2. 使用`QEMU Monitor`命令行
由于之前我们使用串口占用了标准输出, 那如果我们希望使用`QEMU Monitor`命令行, 那么我们需要将串口输出重定向到文件, 这里我们可以使用`-serial file:output.txt`来将串口输出重定向到文件, 这里我们可以将`command`改为:
```json
"command": "echo 'QEMU RTOSdemo started'; <Path>/build/qemu-system-arm -machine mps2-an385 -cpu cortex-m3 -kernel ${workspaceFolder}/build/gcc/output/RTOSDemo.out -monitor stdio -serial file:output.txt -s -S"
```

这两种配置方式各有利弊, 我们目前阶段都还好了.

接下来就是使用`QEMU Monitor`来查看代码中寄存器的值了, 运行`F5`后,开始调试,单步调试到`main.c`中的`prvUARTInit`中:
```c
static void prvUARTInit( void )
{
    UART0_BAUDDIV = 16;
    UART0_CTRL = 1;
}
```
这个时候, 先不要着急进行下一步调试, 在这里先暂停下, 查看下`UART0_BAUDDIV`的定义: 
```c
 * required UART registers. */
#define UART0_ADDRESS                         ( 0x40004000UL )
#define UART0_DATA                            ( *( ( ( volatile uint32_t * ) ( UART0_ADDRESS + 0UL ) ) ) )
#define UART0_STATE                           ( *( ( ( volatile uint32_t * ) ( UART0_ADDRESS + 4UL ) ) ) )
#define UART0_CTRL                            ( *( ( ( volatile uint32_t * ) ( UART0_ADDRESS + 8UL ) ) ) )
#define UART0_BAUDDIV                         ( *( ( ( volatile uint32_t * ) ( UART0_ADDRESS + 16UL ) ) ) )
```
通过这个宏定义, 我们可以得知`UART0_BAUDDIV`的地址为`0x40004010`, 并且数据类型为`uint32_t`, 那么我们可以通过`QEMU Monitor`来查看这个地址的值, 在`QEMU Monitor`中输入`xp /1xw 0x40004010` 得到如下输出:
```shell
(qemu) xp /1xw 0x40004010
0000000040004010: 0x00000000
```
这里的`xp`表示`eXamine Physical memory`, `/1xw`表示`/fmt`, 其中:
- `w` 代表了查看的数据size为`word`, `word`的大小为`4`个字节, 32bits. 
- `x` 代表了查看的数据的格式为`hex`, 即16进制.
- `/1` 代表了查看的数据的个数为`1`, 即查看一个`word`的数据.

这里我们可以看到, `UART0_BAUDDIV`的值为`0x00000000`, 这是因为我们停在了`prvUARTInit`中, 还没有对`UART0_BAUDDIV`进行赋值, 所以这里的值为`0x00000000`, 这里我们可以继续单步调试, 然后再次查看`UART0_BAUDDIV`的值.
```shell
(qemu) xp /1xw 0x40004010
0000000040004010: 0x00000010
```
这里我们可以看到, `UART0_BAUDDIV`的值为`0x00000010`, 这是因为我们在`prvUARTInit`中对`UART0_BAUDDIV`进行了赋值, 这里的值为`0x00000010`, 后续的`UART0_CTRL`也可以通过这种方式来查看, 大家可以自行尝试.

在这里, 再稍微延伸一点, 为什么是`0x40004010`呢? 经常弄单片机的小伙伴肯定理解这是`UART0`的基地址, 而`16` 则是`BAUDDIV` 在`UART0`中的偏移量, 这个值是通过手册查找得到的, 这里就不多赘述了.

最后,我们通过一个简单的`info mtree`来查看内存的布局:
```shell
(qemu) info mtree
(qemu) info mtree
address-space: cpu-memory-0
  0000000000000000-ffffffffffffffff (prio 0, i/o): armv7m-container
...
      0000000040004000-0000000040004fff (prio 0, i/o): uart
...

address-space: bitband-source
address-space: bitband-source
address-space: memory
  0000000000000000-ffffffffffffffff (prio -1, i/o): system
...
    0000000040004000-0000000040004fff (prio 0, i/o): uart
...
```

从内存布局中, 我们也能看到`UART`的地址, 这里就不多赘述了.

# 写在最后/更新频次说明
这次我们简单介绍了一下`FreeRTOS` 中的一个示例, 并由此衍生出了一些小配置及`monitor`中的一些简单命令, 这里只是一个简单的入门, 后续还有很多内容需要去深入了解, 也希望大家能够多多尝试, 多多探索.

后续的更新频次会保证在一周最少一篇, 尽可能的保证质量, 但是由于个人能力有限, 也希望大家能够多多包涵, 如果有什么问题, 也欢迎大家提出, 我会尽力解答.

最后, 感谢大家的阅读, 也希望大家能够多多支持, 谢谢!

下期内容暂时未定, 但是应该和我最近遇到的问题有关, 也希望大家能够多多关注, 谢谢!

# 参考资料
- [FreeRTOS](https://www.freertos.org/)
- [FreeRTOS Github](https://github.com/FreeRTOS/FreeRTOS)
- https://dev.to/iotbuilders/debugging-freertos-with-qemu-in-vscode-4j52
- [QEMU Monitor](https://qemu-project.gitlab.io/qemu/system/monitor.html)