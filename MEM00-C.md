# MEM00-C. Allocate and free memory in the same module, at the same level of abstraction
# 分配和释放内存代码,放在同一模块的同一抽象层次上

动态内存管理是软件中引起安全漏洞的一个常见原因.糟糕的内存管理会导致堆栈溢出、空指针、多次释放等安全问题。从程序员的角度来说，内存管理主要涉及内存的申请、读写和释放。
如果在代码中不同的模块或者不同的抽象层次上分别进行内存的申请和释放，会使得我们难以判断内存是何时释放的，同时难以辨别某一块内存是否已经释放。
为了避免这种情况的处理，理想情况下，内存应该在同一个模块内，并且在同一个抽象层次上进行申请和释放。包括在C标准规范 [ISO/IEC 9899:2011]的7.23.3章节中描述到的以下内存分配和释放函数的使用。

```
void *malloc(size_t size);
 
void *calloc(size_t nmemb, size_t size);
 
void *realloc(void *ptr, size_t size);
 
void *aligned_alloc(size_t alignment, size_t size);
  
void free(void *ptr);
```

## 不合规代码示例

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
