
## 11.环境

### 11.1 ENV30-C.不要修改某些函数的返回值所引用的对象
有些函数返回一个指向某个对象的指针，而修改该对象的值会导致未定义的行为。这些函数包括getenv()，setlocale()，localeconv()，asctime()以及strerror()。在这种情况下，函数调用的结果必须当作const限定来进行对待。
C语言标准 7.22.4.6，段落4[ISO/IEC 9899:2011]对getenv()定义如下：
>getenv()函数返回一个指向一个字符串的指针，该字符串与匹配的列表成员相关。程序不能修改所指向的字符串，但下次调用getenv函数可能会覆盖该字符串。如果无法找到指定的名称，则返回一个null指针。

如果必须要对getenv()返回的字符串进行修改，应该创建一份本地的拷贝。修改getenv()返回的字符串是未定义行为（参见undefined behavior 184。）

类似地，7.11.1.1小节，段落8[ISO/IEC 9899:2011]，对setlocale()定义如下：
>setlocale函数返回指向字符串的指针，后续通过该字符串的值及其关联的类别来调用该函数，可以恢复那一部分程序的地区设置。程序不能修改指针所指向的字符串，但该字符串可能会被后续的setlocale函数调用所覆盖。

同时，7.11.2.1小节，段落8[ISO/IEC 9899:2011]对localeconv()定义如下：
>localeconv函数返回一个指向了已填写了内容的对象的指针。程序不能修改返回值所指向的结构体，但该结构体可能会被后续的localeconv函数调用覆盖。另外，使用类别LC_ALL，LC_MONETARY或LC_NUMERIC来调用setlocale函数可能会覆盖该结构体的内容。

修改setlocale()返回的字符串或者localeconv()返回的结构体是未定义的行为。（参见undefined behaviors 120和121。）此外，C语言标准没有对setlocale()返回的字符串内容做出任何要求。所以，无法对该字符串的内部内容或结构做任何假定。

最后，7.24.6.2小节，段落4[ISO/IEC 9899:2011]声明：
>strerror函数返回一个指向字符串的指针，字符串的内容与特定的区域设置有关。程序不能修改所指向的字符串内容，但可能会被后续的strerror函数调用所覆盖。

修改strerror返回的字符串是未定义行为(参见undefined behavior 184。)

#### 11.1.1 不合规的代码示例(getenv())
此不合规的代码示例修改了getenv()所返回的字符串内容，将所有的双引号(")替换成下划线(_)：

```
#include <stdlib.h>

void trstr(char *c_str, char orig, char rep) {
    while (*c_str != '\0') {
        if (*c_str == orig) {
            *c_str = rep;
        }
        ++c_str;
    }
}
                  
void func(void) {
    char *env = getenv("TEST_ENV");
    if (env == NULL) {
        /* Handle error */
    }
    trstr(env,'"', '_');
}
```

#### 11.1.2 合规的解决方案（getenv()）（未修改环境变量的值）

