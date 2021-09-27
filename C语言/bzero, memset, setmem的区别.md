# bzero, memset, setmem的区别

## bzero函数

原型：

extern void bzero(void *s, int n);

用法：

#include <string.h>

功能：置字节字符串s的前n个字节为零。

说明：bzero无返回值

举例：

```c
#include <syslib.h>
       #include <string.h>
       int main()
       {
                struct
                {
                      int a;
                      char s[5];
                      float f;
                } tt;
                char s[20];
                bzero(&tt,sizeof(tt));   // struct initialization to zero       bzero(s,20);
                clrscr();
                printf("Initail Success");
                getchar();
                return 0;
        }
```

## memset函数

原型：

extern void *memset(void *buffer, int c, int count);

用法：

#include <string.h>

功能：把buffer所指内存区域的前count个字节设置成字符c。

说明：返回指向buffer的指针

举例：

```c
#include <syslib.h>
       #include <string.h>
       int main()
       {
           char *s="Golden Global View";
            clrscr();
           memset(s,'G',6);
           printf("%s",s);
           getchar();
            return 0;
        }
```

## setmem函数

原型：

extern void setmem(void *buf, unsigned int count, char ch);

用法：

#include <string.h>

功能：把buf所指内存区域前count个字节设置成字符ch。

说明：返回指向buf的指针。

举例：

```c
#include <syslib.h>
       #include <string.h>
       int main()
       {
            char *s="Golden Global View";
           clrscr();
           setmem(s,6,'G');
          printf("%s",s);
            getchar();
          return 0;
        }
```

## 综述

bcopy、**bzero**和bcmp是传统BSD的函数，属于POSIX标准；mem*是C90(以及C99)标准的C函数。区别在于，如果你打算把程序弄到一个符合C90/C99，但是不符合POSIX标准的平台时，后者比较有优势。

 NetBSD的代码中有很多地方使用mem*(他们更偏爱mem*，以利于移植)，即使内核也是如此，而FreeBSD的内核中则尽量避免使用(希望尽可能避免在内核中出现较多的C函数)。如果你提交代码的话需要注意这些约定。

 在memset和**bzero**初始化数据间，我很多时候选择**bzero**, memset的一个缺点是第二个参数和第三个参数需要记忆，需要记住哪个是值和哪个是大小（如果不想查手册的话）, 不可以弄错。

 C has memset(), the Berkeley UNIX C library has  **bzero**(). They are not identical, and **bzero**() pre dates memset() but is not widely available (since
 it's not part of standard C).
    在LINUX平台上是支持**bzero**的，但是其并不在ANSI C中定义，也就是不属于C的库函数.