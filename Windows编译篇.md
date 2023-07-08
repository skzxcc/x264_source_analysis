# Windows编译x264

## MSVC编译

Ref：https://www.roxlu.com/2016/057/compiling-x264-on-windows-with-msvc

#### 1、下载msys2

https://www.msys2.org/



打开MYSY2应用(或者打开msys2_shell.cmd)，运行如下命令

```shell
pacman -Sy pacman
pacman -Syu   # 运行完这行命令之后MSYS2窗口回退出，重新打开
pacman -Su

pacman -S nasm  # 如果之前已经安装好了nasm汇编器这里可以不用安装(需要将nasm.exe配置到环境变量中)
pacman -S make
```



#### 2、下载x264源码

http://git.videolan.org/git/x264.git 

或者https://github.com/mirror/x264



#### 3、打开VS2017的命令行工具 VS2017 x64 Native Tools Command Prompt

![](windows编译篇.assets/打开VS命令行工具.png)



进入到msys2的msys2_shell.cmd所在目录下，**运行：msys2_shell.cmd  -full-path命令**

![](windows编译篇.assets/运行msys2.png)



此时会重新打开一个MSYS窗口，此窗口下进入到x264源码所在根目录(configure文件的那个目录)

**这里需要用VS2017的命令行工具去运行上述命令，是为了在后续编译x264能找到cl.exe编译器。 不能直接通过cmd去打开**



运行： **(需要通过CC=cl指定使用MSVC的cl编译器)**

```shell
mkdir bin
CC=cl ./configure --prefix=./bin --enable-static
make
make install
```

完成之后lib、include、bin就会生成到当前的bin目录下





## MinGW编译

#### 1、下载msys2

https://www.msys2.org/

打开MYSY2应用(或者打开msys2_shell.cmd)，运行如下命令

```shell
pacman -Sy pacman
pacman -Syu   # 运行完这行命令之后MSYS2窗口回退出，重新打开
pacman -Su

pacman -S nasm # 如果之前已经安装好了nasm汇编器这里可以不用安装(需要将nasm.exe配置到环境变量中)
pacman -S make
```



#### 2、安装MinGW

注意将MinGW的bin目录配置到环境变量中



#### 3、下载x264源码

http://git.videolan.org/git/x264.git 

或者https://github.com/mirror/x264



#### **4、**打开MinGW

**cmd**进入到msys2的msys2_shell.cmd所在目录下，**运行：“msys2_shell.cmd  -full-path -mingw32” 命令**，此时会新创建一个**MinGW32的窗口**

-full-path (或者-use-full-path)是为了能在mingw中能够访问到windows的环境变量

（这里直接用cmd下运行打开msys2的msys2_shell.cmd即可，不需要通过VS的命令行工具）

#### 5、在MinGW的窗口中进入到x264目录

可以现在MinGW窗口中输入”gcc -v “看是否能找到gcc编译器，如果找不到的话检查下**步骤2中是否将MinGW配置到环境变量中了**，不然后续编译会提示”找不到C编译器“



#### 6、 编译x264

运行： **(这里通过--host=ming32指定使用MinGW32，不然可能会提示”找不到C编译器“)**

```shell
mkdir bin
./configure --prefix=./bin --enable-static --host=mingw32
make
make install
```

完成之后lib、include、bin就会生成到当前的bin目录下



