# gcc的编译过程

## 一、预处理

```markdown
xxx-gcc -E -o hello.i hello.c
```

## 二、编译

```markdown
xxx-gcc -S -o hello.s hello.i
```

## 三、汇编(as命令)

```markdown
xxx-gcc -c -o hello.o hello.s
```

## 四、链接(collect2命令)

```markdown
xxx-gcc -o hello hello.o 其他.o
```

>此过程是命令gcc -o hello hello.c执行过程中的整个过程