# C and gdb

## C pointer
---

指针：a group of cells (bytes, often two or four) that can hold an address

指针的大小比较：只有在两个指针指向同一个数组的元素时比较才合法。指针只有相减是合法的

`char*`和`char []`的区别：前者是指针，并不会管到字符串，后者是数组，需要一个足够大的容量来容纳整个字符串

```c
char amessage[] = "now is the time"; /* an array */
char *message = "now is the time"; /* a pointer, points to the string constant, could not modify the string content */
```

C 程序的内存分配：

1.  堆 (heap)：由程序员通过`malloc()`和`free()`使用和释放，如果忘记释放可能造成内存泄漏。地址从低到高增长
2.  栈 (stack)：编译器自动分配释放，存放函数的参数值、局部变量的值等。在退出函数后自动释放销毁。地址从高到低增长
3.  全局区 (static)：通过`static`声明，全局都可以访问，不会在函数退出后释放

