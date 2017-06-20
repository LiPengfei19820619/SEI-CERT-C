
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
如果程序员不打算修改环境变量的值，此合规的解决方案演示了如何对返回值的一份拷贝进行修改：
```
#include <stdlib.h>
#include <string.h>
void trstr(char *c_str, char orig, char rep) {
    while (*c_str != '\0') {
        if (*c_str == orig) {
            *c_str = rep;
        }
        ++c_str;
    }
}

void func(void) {
    const char *env;
    char *copy_of_env;
    env = getenv("TEST_ENV");
    if (env == NULL) {
        /* Handle error */
    }
    copy_of_env = (char *)malloc(strlen(env) + 1);
    if (copy_of_env == NULL) {
        /* Handle error */
    }
    strcpy(copy_of_env, env);
    trstr(copy_of_env,'"', '_');
    /* ... */
    free(copy_of_env);
}
```

#### 11.1.3 合规的解决方案（getenv()）（使用POSIX接口修改环境变量的值）
如果程序员的意图就是修改环境变脸的值，可以采用此合规的解决方案，它使用POSIX的setenv()和strdup()函数将修改后的字符串存回到环境变量中：
```
#include <stdlib.h>
#include <string.h>
void trstr(char *c_str, char orig, char rep) {
    while (*c_str != '\0') {
        if (*c_str == orig) {
            *c_str = rep;
        }
        ++c_str;
    }
}

void func(void) {
    const char *env;
    char *copy_of_env;

    env = getenv("TEST_ENV");
    if (env == NULL) {
        /* Handle error */
    }

    copy_of_env = strdup(env);
    if (copy_of_env == NULL) {
        /* Handle error */
    }

    trstr(copy_of_env,'"', '_');

    if (setenv("TEST_ENV", copy_of_env, 1) != 0) {
        /* Handle error */
    }

    /* ... */
    free(copy_of_env);
}
```

#### 11.1.4 不合规的代码示例（localconv()）
在此不合规的代码示例中，直接修改了localeconv()返回的对象：
```
#include <locale.h>

void f2(void) {
    struct lconv *conv = localeconv();

    if ('\0' == conv->decimal_point[0]) {
        conv->decimal_point = ".";
    }
}
```

#### 11.1.5 合规的解决方案（localeconv()）（拷贝）
此合规的解决方案修改了localeconv()返回的对象的一份拷贝：
```
#include <locale.h>
#include <stdlib.h>
#include <string.h>

void f2(void) {
    const struct lconv *conv = localeconv();
    if (conv == NULL) {
        /* Handle error */
    }

    struct lconv *copy_of_conv = (struct lconv *)malloc(
        sizeof(struct lconv));
    if (copy_of_conv == NULL) {
        /* Handle error */
    }

    memcpy(copy_of_conv, conv, sizeof(struct lconv));

    if ('\0' == copy_of_conv->decimal_point[0]) {
        copy_of_conv->decimal_point = ".";
    }
    /* ... */
    free(copy_of_conv);
}
```

#### 11.1.6 风险评估



### 11.2 ENV31-C.在执行了某个可能导致环境变量指针无效的操作之后就不要再依赖于该指针
有些实现提供了一个不可移植的环境变量指针，该指针在调用main()时有效，但是可能会因为修改环境变量的操作而导致无效。

C语言标准J.5.1小节[ISO/IEC 9899:2011]规定：
>在托管环境中，main函数接受第三个参数，char *envp[]，指向一个以null终止的字符指针数组，其中的每个指针指向一个字符串，该字符串提供了用于程序本次执行的环境变量的相关信息。

因此，在支持此常见扩展的托管环境下，有可能通过此改良形式的main()来访问环境变量：
```
main(int argc, char *argv[], char *envp[]){ /* ... */ }
```
然而，通过任何手段修改环境变量都可能会导致重新分配存放环境变量值的内存，致使envp引用了错误的位置。例如，如果使用GCC 4.8.1编译并运行于32位GNU/Linux机器上，下面的代码，
```
#include <stdio.h>
#include <stdlib.h>

extern char **environ;

int main(int argc, const char *argv[], const char *envp[]) {
    printf("environ: %p\n", environ);
    printf("envp:    %p\n", envp);
    setenv("MY_NEW_VAR", "new_value", 1);
    puts("--Added MY_NEW_VAR--");
    printf("environ: %p\n", environ);
    printf("envp:    %p\n", envp);
    return 0;
}
```
输出结果如下：
```
% ./envp-environ
environ: 0xbf8656ec
envp:    0xbf8656ec
--Added MY_NEW_VAR--
environ: 0x804a008
envp:    0xbf8656ec
```
从这些结果很明显的看出，由于调用了setenv()，环境变量的存储位置被重新分配了。外部变量environ被更新以引用到当前的环境变量，而envp参数则没有更新。

指向环境变量的指针也可能会由于后续的getenv()调用而失效。（参见ENV34-C.Do not store pointers returned by certain functions。）

#### 11.2.1 不合规的代码示例（POSIX）
在调用POSIX setenv()函数或者其他的修改环境变量的函数之后，envp指针就有可能不再指向当前的环境变量了。《Portable Operating System Interface(POSIX), Base Specifications Issue 7>>[IEEE Std 1003.1:2013]中规定：
>如果setenv()修改了外部变量environ，会导致不可预期的结果。特别是，如果main()的可选参数envp存在，该参数不会被修改，所以可能会指向环境变量的一个过时的拷贝（因为environ可能会存在其他的拷贝）。

此不合格的代码示例在调用了setenv()之后访问了envp指针：
```
#include <stdio.h>
#include <stdlib.h>

int main(int argc, const char *argv[], const char *envp[]) {
    if (setenv("MY_NEW_VAR", "new_value", 1) != 0) {
        /* Handle error */
    }
    if (envp != NULL) {
        for (size_t i = 0; envp[i] != NULL; ++i) {
            puts(envp[i]);
        }
    }
    return 0;
}
```

因为envp可能不再指向当前的环境变量，该程序具有未定义的行为。

#### 11.2.2 合规的解决方案（POSIX）
如果定义了environ，则使用environ来替换envp：
```
#include <stdio.h>
#include <stdlib.h>

extern char **environ;

int main(void) {
    if (setenv("MY_NEW_VAR", "new_value", 1) != 0) {
    /    * Handle error */
    }

    if (environ != NULL) {
        for (size_t i = 0; environ[i] != NULL; ++i) {
            puts(environ[i]);
        }
    }

    return 0;
}
```

#### 11.2.3 不合规的代码示例（Windows）
在调用Windows的_putenv_s()函数或者其他的修改环境变量的函数之后，envp指针就可能不再指向环境变量。
根据Visual C++参考资料[MSDN]：
>传递到main和wmain的环境变量块是当前环境变量的一份“冻结的”拷贝。如果你后续通过调用_putenv或者_wputenv来修改环境变量，当前的环境变量（由getenv/_wgetenv函数返回的值以及_environ/_wenviron变量的值）会发生改变，但是envp指向的内存块的内容则不会发生变化。

此不合规的代码示例在调用_putenv_s()之后访问envp指针：
```
#include <stdio.h>
#include <stdlib.h>

int main(int argc, const char *argv[], const char *envp[]) {
    if (_putenv_s("MY_NEW_VAR", "new_value") != 0) {
        /* Handle error */
    }

    if (envp != NULL) {
        for (size_t i = 0; envp[i] != NULL; ++i) {
            puts(envp[i]);
        }
    }

    return 0;
}
```

因为envp不再指向当前的环境变量，该程序具有未定义的行为。

#### 11.2.4 合规的解决方案（Windows）
此合规的解决方案使用_environ变量来代替envp：
```
#include <stdio.h>
#include <stdlib.h>

_CRTIMP extern char **_environ;

int main(int argc, const char *argv[]) {
    if (_putenv_s("MY_NEW_VAR", "new_value") != 0) {
        /* Handle error */
    }

    if (_environ != NULL) {
        for (size_t i = 0; _environ[i] != NULL; ++i) {
            puts(_environ[i]);
        }
    }

    return 0;
}
```

#### 11.2.5 合规的解决方案
在存在大量的不合规的envp代码时，本合规的解决方案可以减少补救所需的时间。它将
```
int main(int argc, char *argv[], char *envp[]) {
    /* ... */
}
```
替换为
```
#if defined (_POSIX_) || defined (__USE_POSIX)
    extern char **environ;
    #define envp environ
#elif defined(_WIN32)
    _CRTIMP extern char **_environ;
    #define envp _environ
#endif

int main(int argc, char *argv[]) {
    /* ... */
}
```

此合规的解决方案可能需要进行扩展，以支持其他形式的外部变量environ的实现。

#### 11.2.6 风险评估


### 11.3 所有的退出处理程序必须正常返回
C语言标准提供了3个导致应用程序正常终止的函数：_Exit()，exit()，以及quick_exit()。这些函数统称为退出函数。在调用exit()函数，或者控制流程转移到main()入口点函数之外时，会调用由atexit()（而不是at_quick_exit()）所注册的函数。在调用quick_exit()函数时，会调用由at_quick_exit()（而不是atexit)所注册的函数。这些函数统称为退出处理程序。在调用_Exit()函数时，不会调用退出处理程序或者信号处理程序。
退出处理程序必须通过返回来进行终止。对于所有的退出处理程序来说，能够执行他们的清理操作，是很重要的，同时在潜在的安全性方面也是非常关键的。而由于应用程序开发人员并不总是了解由支持库所安装的处理程序，这一点就变得尤其重要了。这方面有两个特定的问题，包括嵌套调用退出函数，和通过调用longjmp来终止退出处理程序的调用。
嵌套调用退出处理函数是未定义的行为。（参见undefined behavior 182。）这种行为只可能发生在从退出处理程序中调用退出函数，或者在信号处理程序中调用退出函数。（参见SIG30-C. Call only asynchronous-safe functions within signal handlers。）
如果longjmp()函数的调用，将会终止由atexit()注册的函数的调用，则该行为是未定义的。

#### 11.3.1 不合规的代码示例
在该不合规的代码示例中，通过atexit()注册了exit1()和exit2()函数，以在程序终止时执行所需的清理操作。然而，如果some_condition运算的结果为真，exit()会被二次调用，产生未定义的行为。

```
#include <stdlib.h>

void exit1(void) {
    /* ... Cleanup code ... */
    return;
}

void exit2(void) {
    extern int some_condition;
    if (some_condition) {
        /* ... More cleanup code ... */
        exit(0);
    }
    return;
}

int main(void) {
    if (atexit(exit1) != 0) {
        /* Handle error */
    }
    if (atexit(exit2) != 0) {
        /* Handle error */
    }
    /* ... Program code ... */
    return 0;
}
```
通过atexit()函数注册的函数，会以注册时的顺序相反的次序进行调用。因此，如果exit2()通过除返回之外的任何方式退出，exit1()就不会被执行。由支持库安装的atexit()处理程序同样如此。

#### 11.3.2 合规的解决方案
通过atexit()注册为退出处理程序的函数必须以返回的方式退出，如下面的合规解决方案：
```
#include <stdlib.h>

void exit1(void) {
    /* ... Cleanup code ... */
    return;
}

void exit2(void) {
    extern int some_condition;
    if (some_condition) {
        /* ... More cleanup code ... */
    }
    return;
}

int main(void) {
    if (atexit(exit1) != 0) {
        /* Handle error */
    }
    if (atexit(exit2) != 0) {
        /* Handle error */
    }
    /* ... Program code ... */
    return 0;
}
```

#### 11.3.3 不合规的代码示例
在该不合规的代码示例中，通过atexit()注册了exit1()，所以在程序终止时会调用exit1()。而exit1()函数跳转回到main()来返回，产生了未定义的结果。
```
#include <stdlib.h>
#include <setjmp.h>

jmp_buf env;
int val;

void exit1(void) {
    longjmp(env, 1);
}

int main(void) {
    if (atexit(exit1) != 0) {
        /* Handle error */
    }
    if (setjmp(env) == 0) {
        exit(0);
    } else {
        return 0;
    }
}
```

#### 11.3.4 合规的解决方案
本合规的解决方案没有调用longjmp()，而是从退出处理程序中正常返回：
```
#include <stdlib.h>

void exit1(void) {
    return;
}

int main(void) {
    if (atexit(exit1) != 0) {
        /* Handle error */
    }
    return 0;
}
```

#### 11.3.5 风险评估


### 11.4 不要调用system()
C语言标准system()函数通过调用实现定义的命令处理器来执行一个指定的命令，例如UNIX的shell，或者Microsoft Windows下的CMD.EXE。POSIX的popen()和Windows的_popen()函数同样调用命令处理器，但是会在调用程序和被执行的命令之间创建一个管道，并返回一个指向流的指针，该指针可用于从管道中读取数据或者向管道写入数据[IEEE Std 1003.1:2013]。

使用system()函数可能会导致可被利用的漏洞，最坏的情况是允许执行任意的系统命令。
调用system()具有高风险的场景如下：


