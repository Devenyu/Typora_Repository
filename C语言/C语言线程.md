# C语言线程

## 线程的创建

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

void* myfunc(void* args)
{
    int i;
    //char* name = (char*) args;
    for(i=1; i<50 i++)
    {
        printf("%d\n", i);
    }
    //printf("Hello World\n");
    return NULL;
}

int main()
{
    pthread_t th1;
    pthread_t th2;
    pthread_create(&th1, NULL, myfunc, NULL);
    pthread_create(&th2, NULL, myfunc, NULL);
    pthread_join(th1, NULL);
    pthread_join(th2, NULL);
    return 0;
}
```

## 往线程中传参数

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

int arr[5000];
int s1 = 0;
int s2 = 0;
    
void* myfunc1(void* args)
{
    int i;
    char* name = (char*) args;
    for(i=1; i<2500; i++)
    {
        s1 = s1 + arr[i];
    }
    //printf("Hello World\n");
    return NULL;
}

void* myfunc2(void* args)
{
    int i;
    //char* name = (char*) args;
    for(i=2500; i<5000  ; i++)
    {
        s2 = s2+ arr[i];
    }
    //printf("Hello World\n");
    return NULL;
}

int main()
{
    int i;
    for(i=0; i<5000; i++)
    {
        arr[i] = rand() % 50;
    }
    for(i=0; i<5000; i++)
    {
        printf("arr[%d] = %d\n", i, arr[i]); 
    }
    pthread_t th1;
    pthread_t th2;
    pthread_create(&th1, NULL, myfunc1, NULL);
    pthread_create(&th2, NULL, myfunc2, NULL);
    pthread_join(th1, NULL);
    pthread_join(th2, NULL);
    
    printf("s1 = %d\n",s1);
    printf("s2 = %d\n",s2);
    printf("s1 + s2 = %d\n", s1 +s2);
    return 0;
}
```

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

typedef struct
{
    int first;
    int last;
    int result;
} MY_ARGS;

int arr[5000];
int s1 = 0;
int s2 = 0;
    
void* myfunc(void* args)
{
    int i;
    int s = 0;
    MY_ARGS* my_args = (MY_ARGS*) args;
    int first = my_args -> first;
    int last = my_args -> last;
    
    for(i=first; i<last; i++)
    {
        s = s + arr[i];
    }
    my_args -> result = s;
    //printf("Hello World\n");
    return NULL;
}

int main()
{
    int i;
    for(i=0; i<5000; i++)
    {
        arr[i] = rand() % 50;
    }
    for(i=0; i<5000; i++)
    {
        printf("arr[%d] = %d\n", i, arr[i]); 
    }
    pthread_t th1;
    pthread_t th2;
    MY_ARGS args1 = {0, 2500, 0};
    MY_ARGS args2 = {2500, 5000, 0};
    
    pthread_create(&th1, NULL, myfunc, &args1);
    pthread_create(&th2, NULL, myfunc, &args2);
    pthread_join(th1, NULL);
    pthread_join(th2, NULL);

    s1 = args1.result;
    s2 = args2.result;
    printf("s1 = %d\n", s1);
    printf("s2 = %d\n", s2);
    printf("s1 + s2 = %d\n", s1+s2);
    
    return 0;
}
```



## race condition和锁的应用



```
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

typedef struct
{
    int first;
    int last;
} MY_ARGS;

int arr[5000];
int s = 0;

pthread_mutex_t lock;

void* myfunc(void* args)
{
	int i = 0;
	for(i=0; i<1000000; i++)
	{
		pthread_mutex_lock(&lock); 
		s++;
		pthread_mutex_unlock(&lock);
	}
	return NULL;
}

int main()
{
    
    pthread_t th1;
    pthread_t th2;
    pthread_mutex_init(&lock, NULL);
    
    pthread_create(&th1, NULL, myfunc, NULL);
    pthread_create(&th2, NULL, myfunc, NULL);
    pthread_join(th1, NULL);
    pthread_join(th2, NULL);
    
    printf("s1 = %d\n",s1);
    printf("s2 = %d\n",s2);
    printf("s1 + s2 = %d\n", s1 +s2);
    return 0;
}
```

## false Sharing

![image-20210811150955960](http://qyateyap7.hn-bkt.clouddn.com/img/image-20210811150955960.png)

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

#define MAX_SIZE 50000000

typedef struct
{
    int first;
    int last;
    int id;
} MY_ARGS;

int* arr;
int results[2];
    
void* myfunc(void* args)
{
    int i;
    MY_ARGS* my_args = (MY_ARGS*) args;
    int first = my_args -> first;
    int last = my_args -> last;
    int id = my_args -> id;
    
    for(i=first; i<last; i++)
    {
        results[id] = results[id] + arr[i];
    }
    //printf("Hello World\n");
    return NULL;
}

int main()
{
    int i;
    arr = malloc(sizeof(int) * MAXSIZE);
    for(i=0; i<MAX_SIZE; i++)
    {
        arr[i] = rand() % 5;
    }
    results[0] = 0;
    results[1] = 0;
    
    pthread_t th1;
    pthread_t th2;
    
    int mid = MAX_SIZE / 2; 
    MY_ARGS args1 = {0, mid, 0};
    MY_ARGS args2 = {mid, MAX_SIZE, 1};
    
    pthread_create(&th1, NULL, myfunc, &args1);
    pthread_create(&th2, NULL, myfunc, &args2);
    pthread_join(th1, NULL);
    pthread_join(th2, NULL);

    printf("s1 = %d\n", results[0]);
    printf("s2 = %d\n", results[1]);
    printf("s1 + s2 = %d\n", results[0] + results[1]);
    
    return 0;
}
```

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

#define MAX_SIZE 50000000

typedef struct
{
    int first;
    int last;
    int id;
} MY_ARGS;

int* arr;
int results[101];
    
void* myfunc(void* args)
{
    int i;
    MY_ARGS* my_args = (MY_ARGS*) args;
    int first = my_args -> first;
    int last = my_args -> last;
    int id = my_args -> id * 100;
    
    for(i=first; i<last; i++)
    {
        results[id] = results[id] + arr[i];
    }
    //printf("Hello World\n");
    return NULL;
}

int main()
{
    int i;
    arr = malloc(sizeof(int) * MAXSIZE);
    for(i=0; i<MAX_SIZE; i++)
    {
        arr[i] = rand() % 5;
    }
    results[0] = 0;
    results[100] = 0;
    
    pthread_t th1;
    pthread_t th2;
    
    int mid = MAX_SIZE / 2; 
    MY_ARGS args1 = {0, mid, 0};
    MY_ARGS args2 = {mid, MAX_SIZE, 1};
    
    pthread_create(&th1, NULL, myfunc, &args1);
    pthread_create(&th2, NULL, myfunc, &args2);
    pthread_join(th1, NULL);
    pthread_join(th2, NULL);

    printf("s1 = %d\n", results[0]);
    printf("s2 = %d\n", results[100]);
    printf("s1 + s2 = %d\n", results[0] + results[100]);
    
    return 0;
}
```

