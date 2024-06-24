​
# 代码下载
官方的源码路径为https://github.com/qemu/qemu ,下载的方式有很多, 使用`git clone` 或者直接下载对应版本的代码. 但是由于`github` 的下载速度一言难尽, 所以可以找些加速的方式, 这里只简单说明下, 在使用`git clone` 的时候可以增加`--depth=1` 的方式避免下载过多我们不关注的信息.

如果是需要模拟或者`risc-v` ,也可以在https://github.com/riscv-mcu/qemu/ 中进行下载. 

# 编译
## Windows 平台下的编译
这里只简单提下其中一种方法:
1. 下载[`MSYS2` ](https://www.msys2.org/). 并打开`MSYS2 MINGW64` 的控制台.
2. 更新`repository`:  `pacman -Syu`
3. 安装对应的依赖,这个是取决于具体希望QEMU 支持的功能. 可以参考: https://wiki.qemu.org/Hosts/W32#Native_builds_with_MSYS2
4. 切换到代码目录:  `cd /<盘符,C,D等>/路径`
5. 运行`configure`, 具体的参数需要根据具体的需求, 这里给一个简单的示例: `./configure --enable-debug --target-list=arm-softmmu  --disable-docs`  这里指的是可以只编译`arm` 相关的内容, 如果不指定`target-list`的话, 默认编译的是所有平台的 ,这样会花费比较多的时间.
6. 运行`make`. 

这里只提到了一种很简单的编译的方法, 这个过程中可能还会遇到一些问题, 而且大多数人会选择  先创建`build` 目录后在`build` 目录下处理相关的内容.

如果遇到问题的话,可以留言,看到的话一定第一时间回复.
## Mac 下的编译
Mac 下的编译其实与Linux 下的编译类似, 都比较简单,不同的只不过是包管理工具使用的是`brew` 而已. 具体的可以参考:
https://wiki.qemu.org/Hosts/Mac#Building_QEMU_for_macOS

# 写在最后/下一篇更新预告
太久没有记录了,发现好像自己没有啥可以记录的了. 其实更多的还是自己并没有养成记录的习惯吧. 

下一篇会讲下使用VSCode 配置QEMU的 运行和调试环境.

----
# 参考资料
- 源码路径: https://github.com/qemu/qemu
- Windows 下编译: https://wiki.qemu.org/Hosts/W32
- Mac 下编译: https://wiki.qemu.org/Hosts/Mac

 

​