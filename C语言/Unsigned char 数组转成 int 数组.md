# Unsigned char 数组转成 int 数组

由于在64位编译器中char占1个字节，int占4个字节，所以我们需要将4个char类型的数组赋给int。

我试想过两种方案，一种是将第四个char类型的数据向左移动24位，第三个char类型的数据向左移动16位，第二个char类型的数据向左移动8位，这样把四个数据拼接起来组成一个int类型的数据。

```c
unsigned short dataSize = 12;
unsigned char a[22] = {0x23, 0x45, 0x67, 0x89, 0x10, 0x23, 0x45, 0x23, 0x45, 0x67, 0x89, 0x10};
int b[4];
for (int i = 0; i < (dataSize / 4 + 1); i++)
    {
        if (dataSize % 4 == 0)
        {
            b[i] = a[i * 4] | a[i * 4 + 1] << 8 | a[i * 4 + 2] << 16 | a[i * 4 + 3] << 24;
        }
        else if (dataSize % 4 == 1)
        {
            b[i] = a[i * 4];
        }
        else if (dataSize % 4 == 2)
        {
            b[i] = a[i * 4] | a[i * 4 + 1] << 8;
        }
        else if (dataSize % 4 == 3)
        {
            b[i] = a[i * 4] | a[i * 4 + 1] << 8 | a[i * 4 + 2] << 16;
        }
        printf("b[%d]: %02x\n", i, b[i]);
    }
```

这种做法显然是很蠢的，需要考虑到数据的大小是否能整除4，所以代码特别冗余。

下面介绍一下第二种想法，直接利用memcpy将4位char类型数据赋予int类型的数据，这样即使数据长度不能整除4，编译器只会赋0。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void exchange(unsigned char *Temp, int *val,unsigned short dataSize)
{
    for(int i = 0; i <= dataSize / 4; i++)
    {
        memcpy(val, Temp, 4);
        val++;
        Temp += 4;
    }
}
void main()
{
    unsigned short dataSize = 12;
    unsigned char a[22] = {0x23, 0x45, 0x67, 0x89, 0x10, 0x23, 0x45, 0x23, 0x45, 0x67, 0x89, 0x10};
    int b[4];
    exchange(a, b, dataSize);
    printf("%d\n", dataSize/4);
    for(int i = 0; i < 4; i++)
    {
        printf("%x\n", b[i]);
    }
}
```

运行结果如下：

![image-20210827133428738](http://qyateyap7.hn-bkt.clouddn.com/img/image-20210827133428738.png)

