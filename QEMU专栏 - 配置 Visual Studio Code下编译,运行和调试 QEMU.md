# 写在最前
QEMU 在启动时会涉及到许多配置,  本专栏到目前为止,还是属于新手阶段, 所以这个文档只会涉及到相对简单的内容.  本文档分为两部分:
1. 写在最前的TL;DR , 允许你在拥有镜像文件的基础之上,可以不需要看过多的细节直接将配置复制到自己的环境并进行调整. (可以,但是不推荐)
2. 会涉及有哪些配置, 并且会说明这些

## TL;DR 完整配置
启动相关的配置, `.vscode/launch.json`: 
```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Launch QEMU - test",
            "type": "cppdbg",
            "request": "launch",
            "cwd": "${workspaceRoot}",
            "program": "${workspaceRoot}/build/qemu-system-arm",
            "args": [
                "-machine",
                "mps2-an385", // 指定机器
                "-cpu",
                "cortex-m3", // 指定cpu
                "-nographic", // 指定不需要显示窗口
                "-monitor",
                "none", // 暂时不需要用到monitor 相关的命令,所以不显示.
                "-serial",
                "stdio", // 串口指定到控制台.
                "-kernel",
                "FreeRTOS/Demo/CORTEX_MPS2_QEMU_IAR_GCC/build/gcc/output/RTOSDemo.out"
            ],
            "osx": {
                "MIMode": "lldb"
            },
            "windows": {
                "MIMode": "gdb",
                "miDebuggerPath": "C:\\msys64\\mingw64\\bin\\gdb.exe",
                "setupCommands": [
                    {
                        "description": "Enable pretty-printing for gdb",
                        "text": "-enable-pretty-printing",
                        "ignoreFailures": true
                    },
                    {
                        "description": "Set disassembly-flavor intel",
                        "text": "set disassembly-flavor intel",
                        "ignoreFailures": true
                    }
                ],
            }
            "preLaunchTask": "make" // 在运行之前,先运行vscode的make 任务.
        }
    ]
}
```
任务配置文件, `.vscode/tasks.json` :
```json
{
    "version": "2.0.0",
    "tasks": [
        // 执行make
        {
            "label": "make",
            "type": "shell",
            "windows": {
                "command": "C:\\msys64\\msys2_shell.cmd",
                "args": [
                    "-defterm",
                    "-here",
                    "-no-start",
                    "-mingw64",
                    "-c",
                    "make -j4"
                ],
            },
            "osx": {
                "command": "make",
                "args": [
                    "-j4"
                ],
            },
            "options": {
                "cwd": "${workspaceFolder}"
            },
            "problemMatcher": [
                "$gcc"
            ],
            "group": "build",
            "detail": "Compile QEMU emulator."
        },
        // configure 相关的任务
        {
            "type": "shell",
            "label": "configure",
            "group": "build",
            "windows": {
                "command": "C:\\msys64\\msys2_shell.cmd",
                "args": [
                    "-defterm",
                    "-here",
                    "-no-start",
                    "-mingw64",
                    "-c",
                    "rm -rf build && ./configure --enable-debug --target-list=arm-softmmu --disable-docs"
                ],
            },
            "osx": {
                "command":  "rm -rf build && ./configure --enable-debug --target-list=arm-softmmu --disable-docs"
            },
        }
    ]
}


```

# Visual Studio Code 相关配置
首先使用 `vscode` 打开之前`clone` 下的文件夹,  首先需要创建`launch.json`.

## 增加`launch.json`
首先明确目标, 我们需要通过`vscode`的侧边栏中的`Run And Debug` 来启动QEMU, 首先增加`launch.json`  (完整配置请查看最上面的TL;DR)
```json
// .... 其他配置
{
            "name": "Launch QEMU",
            "type": "cppdbg",
            "request": "launch",
            "cwd": "${workspaceRoot}",
            "program": "${workspaceRoot}/build/qemu-system-arm.exe",
}
```

通过这种方式我们就已经确认了一个最简单的启动配置, 这里简单剖析一下这些参数 :
首先是最基础的 `name`, `type`, 以及`request` , `name`可以自己发挥, 而`type` 和 `request` 则是固定的.  
这里对于`request` 可以再多一句, 它支持`launch` 以及`attach`两种形式:  

 - `launch`: 即会运行对应的`program`.
 - `attach`: 这种是属于附加的形式去调试程序, 即程序已经启动.

`program` 表示需要运行的程序, 这里指的就是你需要调试的应用.
`cwd` 表示当前工作目录, 这个决定了你是从哪启动你的应用的, 也会影响到你 程序中读取文件的相对路径. 一般使用`${workspaceRoot}` 即可,这个环境变量代表vscode 当前工作区间的根目录.
运行 `Launch QEMU` 后, 会得到这样的输出:
```text
qemu\build\qemu-system-arm.exe: No machine specified, and there is no default
Use -machine help to list supported machines
```
从输出上其实可以看出来, 和我们直接运行`program` 并没有太多的区别.

但是这个简单的配置肯定是不符合我们的实际需求, 因此我们需要增加一些参数让 QEMU可以正常运行.
```json

{
// .... 基础配置
            "program": "${workspaceRoot}/build/qemu-system-arm",
             "args": [
                "-machine", "mps2-an385", // 指定机器
                "-cpu", "cortex-m3", // 指定cpu
                "-nographic", // 指定不需要显示窗口
                "-monitor", "none", // 暂时不需要用到monitor 相关的命令,所以不显示.
                "-serial", "stdio", // 串口指定到控制台.
                "-kernel", "FreeRTOS/Demo/CORTEX_MPS2_QEMU_IAR_GCC/build/gcc/output/RTOSDemo.out"
            ],
}
```
可以通过增加`args` 的方式进行增加参数,  这种方式实际上等同于直接运行如下命令:
```shell
${workspaceRoot}/build/qemu-system-arm -machine mps2-an385 -cpu cortex-m3 -nographic -monitor none -serial stdio -kernel FreeRTOS/Demo/CORTEX_MPS2_QEMU_IAR_GCC/build/gcc/output/RTOSDemo.out
```
到这一步的话, 有的人可能就已经可以开始调试, 但是有的人还不行,有如下两个原因:

- 操作系统. 在`Mac` 下需要使用`lldb` ,而在`Windows` 以及`Linux`下使用的都是`gdb`.
- 环境变量. 默认的`gdb` 或者`lldb` 的可执行程序在`PATH`中. 

当然为了保险起见,建议增加如下配置:
```json
{
// .... 上面的基础配置.
	"osx": {
        "MIMode": "lldb"
    },
    "windows": {
        "MIMode": "gdb",
        "miDebuggerPath": "C:\\msys64\\mingw64\\bin\\gdb.exe",
        "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                },
                {
                    "description": "Set disassembly-flavor intel",
                    "text": "set disassembly-flavor intel",
                    "ignoreFailures": true
                }
            ],
    }
}
```
可以通过`windows`以及`osx` 的方式区别不同的系统. 

到这一步可以说最基础的配置已经完成了, 但是这种时候,如果我们修改了QEMU中的代码,并希望继续调试,就比较麻烦,所以这个时候就需要引入任务了.
## 增加`tasks.json` 配置任务
为了确保我们调试的程序是最新代码编译的, 所以在每次调试前,最好确保每次启动的程序和当前的程序是一致的.为此, 我们首先需要增加`make` 用于编译:
```json
{
	"version": "2.0.0",
	"tasks": [
        // 执行make
		{
			"label": "make",
            "type": "shell",
            
            "windows": {
			    "command": "C:\\msys64\\msys2_shell.cmd",
                "args": [
                    "-defterm",
                    "-here",
                    "-no-start",
                    "-mingw64"    , "-c", "make -j4"
                ],
            },
            "osx": {
                "command": "make",
                "args": [
                    "-j4"
                ],
            },
			
			"options": {
				"cwd": "${workspaceFolder}"
			},
			"problemMatcher": [
				"$gcc"
			],
			"group": "build",
			"detail": "Compile QEMU emulator."
		},
	]
}
```
这里使用`msys2` 的 参数, 具体参数含义这里不多描述, 可以使用`C:\\msys64\\msys2_shell.cmd --help ` 进行查看.

配置完任务后, 接下来就是将任务与启动配置关联,  可以通过在`launch.json` 的对应配置中加入如下属性:
```json
"preLaunchTask": "make"
```
具体位置可以参考`TL;DR`. 

另外, 如果觉得想要重新`configure`的话,也可以增加对应的配置, 

```json

        // configure 相关的任务
        {
            "type": "shell",
            "label": "configure",
            "group": "build",
            "windows": {
                "command": "C:\\msys64\\msys2_shell.cmd",
                "args": [
                    "-defterm",
                    "-here",
                    "-no-start",
                    "-mingw64",
                    "-c",
                    "rm -rf build && ./configure --enable-debug --target-list=arm-softmmu --disable-docs"
                ],
            },
            "osx": {
                "command":  "rm -rf build && ./configure --enable-debug --target-list=arm-softmmu --disable-docs"
            },
        }
```

这与`make` 的方法基本一致, 只不过是这个任务平时不怎么运行, 如果需要运行的话,可以通过`Run Task`的方式进行运行.

---
# 写在最后/ 更新预告
写博客终究还是少了点, 目前用的是CSDN自带的Markdown 编辑器, 保存的草稿没能正确保存, 丢了一些内容,再写的时候,总感觉有点词不达意. 但是终于还是写出来了.  下一遍讲下如何使用QEMU调试FreeRTOS. 这个是从国外的一片文章上看到,因为我的目标最终就是去模拟这种嵌入式设备,所以感觉还挺有趣的.

--- 
# 参考资料
- https://code.visualstudio.com/docs/cpp/launch-json-reference
- https://code.visualstudio.com/Docs/editor/debugging
- https://code.visualstudio.com/Docs/editor/tasks
- https://code.visualstudio.com/docs/editor/tasks-appendix