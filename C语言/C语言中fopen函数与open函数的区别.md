# C语言中fopen函数与open函数的区别

- open是系统调用，返回的是文件句柄，文件的句柄是文件在文件描述副表里的索引；fopen是C的库函数，返回的是一个指向文件结构的指针，在不同的系统中应该调用不同的内核api。

- fopen函数可移植，open函数不可以

- fopen函数在用户态下有缓存，所以在进行read和write的时候减少了用户态和内核态的切换，而open函数则每次都需要进行内核态和用户态的切换。如果顺序访问文件，fopen函数要比直接调用open函数快；如果随机访问文件open要比fopen快。

- open函数配合read、write等使用，fopen函数配合fread、fwrite等使用

- ==由于open函数与fopen函数获得的返回类型不同，所以会对后期的文件操作有所影响：==

  open函数的定义如下：

  ```c
    int open(const char * pathname, int flags);
  ```

  ```json
  参数 pathname 指向欲打开的文件路径字符串. 下列是参数flags 所能使用的旗标:
  O_RDONLY 以只读方式打开文件
  O_WRONLY 以只写方式打开文件
  O_RDWR 以可读写方式打开文件. 上述三种旗标是互斥的, 也就是不可同时使用, 但可与下列的旗标利用OR(|)运算符组合.
  O_CREAT 若欲打开的文件不存在则自动建立该文件.
  O_EXCL 如果O_CREAT 也被设置, 此指令会去检查文件是否存在. 文件若不存在则建立该文件, 否则将导致打开文件错误. 此外, 若O_CREAT 与O_EXCL 同时设置, 并且欲打开的文件为符号连接, 则会打开文件失败.
  O_NOCTTY 如果欲打开的文件为终端机设备时, 则不会将该终端机当成进程控制终端机.
  O_TRUNC 若文件存在并且以可写的方式打开时, 此旗标会令文件长度清为0, 而原来存于该文件的资料也会消失.
  O_APPEND 当读写文件时会从文件尾开始移动, 也就是所写入的数据会以附加的方式加入到文件后面.
  O_NONBLOCK 以不可阻断的方式打开文件, 也就是无论有无数据读取或等待, 都会立即返回进程之中.
  O_NDELAY 同O_NONBLOCK.
  O_SYNC 以同步的方式打开文件.
  O_NOFOLLOW 如果参数pathname 所指的文件为一符号连接, 则会令打开文件失败.
  O_DIRECTORY 如果参数pathname 所指的文件并非为一目录, 则会令打开文件失败。
  ```

  ==用open函数打开文件，如果原文件中存在内容，不仅可以在文件尾部追加内容，也可以通过lseek函数将输入的位置定位到文件头部。==具体操作如下：

  ```c
  #include <stdio.h>
  #include <fcntl.h> 
  #include <io.h>
  #include <string.h>
  
  int main()
  {
      char msg[] = "Hello world";
      int fd;
      
      /* 以可读写且文件不存在则自动建立的方式打开文件 */
      fd = open("D:/Devenyu/1.txt", O_RDWR | O_CREAT);
      /* lseek定义如下： 
      off_t lseek(int filedes, off_t offset, int whence);
      参数 offset 的含义取决于参数 whence：
      1. 如果 whence 是 SEEK_SET，文件偏移量将被设置为 offset。
      2. 如果 whence 是 SEEK_CUR，文件偏移量将被设置为 cfo 加上 offset，offset 可以为正也可以为负。
      3. 如果 whence 是 SEEK_END，文件偏移量将被设置为文件长度加上 offset，offset 可以为正也可以为负。*/
      /* 将输入位置定位到文件头部 */
      lseek(fd, 0, SEEK_SET);
      write(fd, msg, sizeof(msg));
      close(fd);
      return 0;
  }
  ```

  fopen函数的定义如下：

  ```c
  FILE *fopen(const char *path, const char *mode);
  ```

  > mode指文件打开的模式

  - r：只读方式打开一个文本文件（该文件必须存在）
  - r+：可读可写方式打开一个文本文件（该文件必须存在）
  - w：只写方式打开一个文本文件（若文件存在则文件长度清为0，即该文件内容会消失。若文件不存在则建立该文件）
  - w+：可读可写方式创建一个文本文件（若文件存在则文件长度清为零，即该文件内容会消失。若文件不存在则建立该文件）
  - a：追加方式打开一个文本文件（若文件不存在，则会建立该文件，如果文件存在，写入的数据会被加到文件尾，即文件原先的内容会被保留。（EOF符保留））
  - a+：可读可写追加方式打开一个文本文件（若文件不存在，则会建立该文件，如果文件存在，写入的数据会被加到文件尾后，即文件原先的内容会被保留。 （原来的EOF符不保留））

  > a 和 a+ 的区别：a 不能读，a+ 可以读

  - rb：只读方式打开一个二进制文件（使用法则同r）
  - rb+：可读可写方式打开一个二进制文件（使用法则同r+）
  - wb：只写方式打开一个二进制文件（使用法则同w）
  - wb+：可读可写方式生成一个二进制文件（使用法则同w+）
  - ab：追加方式打开一个二进制文件（使用法则同a）
  - ab+：可读可写方式追加一个二进制文件（使用法则同a+）

==用fopen打开的函数进行写操作，则只能使用"w+"将原文件内容清除重新写入，或者使用"a+"对原文件内容的末尾进行追加操作，并不能如open函数那般将输入位置前移。==