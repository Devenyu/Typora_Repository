# Makefile的编写

举一个库文件的Makefile实例来详细介绍：

```makefile
#CC = gcc
CC = aarch64-linux-gnu-gcc 
#arm-xilinx-linux-gnueabi-gcc
AR = aarch64-linux-gnu-ar 
#AR = ar
CFLAGS = -std=gnu99 
#CFLAGS = -std=c99 
ROOTDIR = $(PWD)
CFLAGS += -I ../user_api -I ../include

all: 
	$(CC) $(CFLAGS) -fPIC -c ssd_lib.c
	$(CC) -shared -fPIC -o libssd_lib.so ssd_lib.o
	$(AR) r libssd_lib.a ssd_lib.o 

install:
#	cp -f libssd_lib.so /usr/lib/

clean:
	rm -f *.o *.so

```

- CC = aarch64-linux-gnu-gcc表示指定了一个交叉编译的工具。

- CFLAGS = -std=gnu99 中CFLAGS表示用于指定头文件(.h)的路径，如：CFLAGS=-I/usr/include -I/path/include 。

> 拓展->C编译器中的其他选项
>
> ​	LDFLAGS：gcc等编译器会用到的一些优化参数，也能够在里面指定库文件的位置。
>
> ​	使用方法：LDFLAGS=-L/usr/lib -L/path/to/your/lib
>
> ​	LIBS：告诉链接器要链接哪些库文件。
>
> ​	使用方法：LIBS = -lpthread -liconv

- ROOTDIR = $(PWD)是指显示当前所在的目录

> 拓展->(PWD)与(shell pwd)的区别：
>
> ​	P=$(shell pwd)	#这样可以输出路径
>
> ​	P=$(pwd)	#这样没有输出

- -fPIC 作用于编译阶段，告诉编译器产生与位置无关代码(Position-Independent Code)，则产生的代码中，没有绝对地址，全部使用相对地址，故而代码可以被加载器加载到内存的任意位置，都可以正确的执行。这正是共享库所要求的，共享库被加载时，在内存的位置不是固定的。

> 如果不加-fPIC，则加载.so文件的代码段时，代码段引用的数据对象需要重定位，重定位会修改代码段的内容，这就造成每个使用这个.so文件代码段的进程在内核里都会生成这个.so文件代码段的copy。每个copy都不一样，取决于这个.so文件代码段和数据段内存映射的位置，也就是不加fPIC编译出来的so，是要**再加载时根据加载到的位置再次重定位的**。
>
> 但是不用fPIC编译so并不总是不好的，如果你满足以下4个需求：
>
> 1.该库可能需要经常更新
> 		2.该库需要非常高的效率(尤其是有很多全局量的使用时)
> 		3.该库并不很大
> 		4.该库基本不需要被多个应用程序共享

- $(CC) -shared -fPIC -o libssd_lib.so ssd_lib.o	通过ssd_lib.o产生一个libssd_lib.so的共享库。
- $(AR) r libssd_lib.a ssd_lib.o 表示将ssd_lib.o文件插入libssd_lib.a中

> linux ar命令用法
>
> ​	指令参数：
>
> -d 　删除库文件中的成员文件。
> 		-m 　变更成员文件在库文件中的次序。
> 		-p 　显示库文件中的成员文件内容。
> 		-q 　将问家附加在库文件末端。
> 		-r 　将文件插入库文件中。
> 		-t 　显示库文件中所包含的文件。
> 		-x 　自库文件中取出成员文件。
>
> ​	选项参数：
>
> a<成员文件> 　将文件插入库文件中指定的成员文件之后。
> 		b<成员文件> 　将文件插入库文件中指定的成员文件之前。
> 		c 　建立库文件。
> 		f 　为避免过长的文件名不兼容于其他系统的ar指令指令，因此可利用此参数，截掉要放入库文件中过长的成员文件名称。
> 		i<成员文件> 　将问家插入库文件中指定的成员文件之前。
> 		o 　保留库文件中文件的日期。
> 		s 　若库文件中包含了对象模式，可利用此参数建立备存文件的符号表。
> 		S 　不产生符号表。
> 		u 　只将日期较新文件插入库文件中。
> 		v 　程序执行时显示详细的信息。
> 		V 　显示版本信息。
>
> ar用来管理一种文档。这种文档中可以包含多个其他任意类别的文件。这些被包含的文件叫做这个文档的成员。ar用来向这种文档中添加、删除、解出成员。成员的原有属性（权限、属主、日期等）不会丢失。	

以下是test应用程序的Makefile实例：

```makefile
# Makefile

CC = gcc
# CC = aarch64-linux-gnu-gcc
#arm-xilinx-linux-gnueabi-gcc
CFLAGS  = -fPIC -Wall -O -c -g -I../user_api -Wimplicit-function-declaration
TARGET  = ssd_app
OBJS    = ssd_app.o
SRCS    = ssd_app.c

ifeq ($(DEBUG),y)
    DBGFLAGS = -D_PCIE_DEBUG
else
    DBGFLAGS =
endif
    CFLAGS += $(DBGFLAGS)
    
### rules ###
$(TARGET): $(OBJS)
#	$(CC) -g -L ../user_api -o  $(TARGET) $(OBJS) -lssd_lib -lpthread
	$(CC) -g -L ../user_api -o  $(TARGET) $(OBJS) libssd_lib.a -lpthread

$(OBJS) : $(SRCS)
	$(CC) $(CFLAGS) $(SRCS)

clean:
	rm -f $(OBJS) $(TARGET) 

```

- ifeq ($(DEBUG),y)表示是否需要Debug，如果需要则在CFLAGS中加上DEBUG选项。
- \$(TARGET): \$(OBJS)和\$(OBJS) : \$(SRCS)表示TARGET的实行是在OBJS实现的基础上实现的，我们需要先根据原文件生成.o文件，再生成需要的应用程序。