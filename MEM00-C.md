# MEM00-C. Allocate and free memory in the same module, at the same level of abstraction
# 分配和释放内存代码,放在同一模块的同一抽象层次上

动态内存管理是软件中引起安全漏洞的一个常见原因.糟糕的内存管理会导致堆栈溢出、空指针、多次释放等安全问题。从程序员的角度来说，内存管理主要涉及内存的申请、读写和释放。  
如果在代码中不同的模块或者不同的抽象层次上分别进行内存的申请和释放，会使得我们难以判断内存是何时释放的，同时对某一块内存是否已经释放感到困惑。
为了避免这种情况的处理，理想情况下，内存应该在同一个模块内，并且在同一个抽象层次上进行申请和释放。  
译者按【小李飞刀】：所谓的同一个抽象层级，通俗的意义来说，可以理解为在同一个函数中。按照常规，我们将一个函数中的子函数封装，称为一个抽象层级。  
内存分配和释放函数的使用, 在C标准规范 [ISO/IEC 9899:2011]的7.23.3章节中描述到的有以下几个。在实际的项目中，可能会有其他封装或者自定义的内存管理函数，也需要一并考虑。

```
void *malloc(size_t size);
 
void *calloc(size_t nmemb, size_t size);
 
void *realloc(void *ptr, size_t size);
 
void *aligned_alloc(size_t alignment, size_t size);
  
void free(void *ptr);
```

## 不合规代码示例

该不合规的代码示例展示的双释放漏洞，就是由于在不同的抽象层次上进行内存的申请和释放而导致的。该例中，在process_list()函数中为list数组申请了内存。然后该数组被传给了verify_size()函数，此函数对list的大小进行了错误检查。如果list的大小低于某个最小值，则会释放list的内存，然后函数返回。而调用函数对同一块内存又进行了一次释放。

```c
enum { MIN_SIZE_ALLOWED = 32 };
 
int verify_size(char *list, size_t size) {
  if (size < MIN_SIZE_ALLOWED) {
    /* Handle error condition */
    free(list);
    return -1;
  }
  return 0;
}
 
void process_list(size_t number) {
  char *list = (char *)malloc(number);
  if (list == NULL) {
    /* Handle allocation error */
  }
 
  if (verify_size(list, number) == -1) {
      free(list);
      return;
  }
 
  /* Continue processing list */
 
  free(list);
}
```

## 合规的解决方案

为了修正这个问题，需要修改verify_size()函数中的错误处理代码，不再释放list的内存。此修改确保了list的内存仅仅在同一个抽象层次上，也就是在process_list()函数中释放一次。

```
enum { MIN_SIZE_ALLOWED = 32 };
 
int verify_size(const char *list, size_t size) {
  if (size < MIN_SIZE_ALLOWED) {
    /* Handle error condition */
    return -1;
  }
  return 0;
}
 
void process_list(size_t number) {
  char *list = (char *)malloc(number);
 
  if (list == NULL) {
    /* Handle allocation error */
  }
 
  if (verify_size(list, number) == -1) {
      free(list);
      return;
  }
 
  /* Continue processing list */
 
  free(list);
}
```
