
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
        /* Handle error */
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
* 传递的命令字符串来自于受污染的源，而该字符串没有经过合法性检查或者检查不当
* 命令没有指定路径名称，而攻击者可以访问命令处理器路径名称解析机制
* 指定了可执行程序的相对路径，而攻击者有当前工作目录的控制权限
* 攻击者可以伪造指定的可执行程序

不要通过system()或其他类似的函数来调用命令处理器执行命令。

#### 10.4.1 不合规的代码示例
在此不合规的代码示例中，使用了system()函数在主机环境中执行any_cmd。调用命令处理器不是必需的。
```
#include <string.h>
#include <stdlib.h>
#include <stdio.h>

enum { BUFFERSIZE = 512 };

void func(const char *input) {
    char cmdbuf[BUFFERSIZE];
    int len_wanted = snprintf(cmdbuf, BUFFERSIZE,
                              "any_cmd '%s'", input);
    if (len_wanted >= BUFFERSIZE) {
        /* Handle error */
    } else if (len_wanted < 0) {
        /* Handle error */
    } else if (system(cmdbuf) == -1) {
        /* Handle error */
    }
}
```

如果此段代码在Linux系统上编译，并以提升后的权限进行运行，则攻击者可以通过输入下面的字符串来创建一个账户：
```
happy'; useradd 'attacker
```
shell会将上面的字符串解释为两条单独的命令：
```
any_cmd 'happy';
useradd 'attacker'
```
并创建一个新的用户账户，攻击者可以使用此账户来访问被入侵的系统。
此不合规的代码示例同时也违背了 STR02-C. Sanitize data passed to complex subsystems。

#### 11.4.2 合规的解决方案（POSIX)
在此合规的解决方案中，用execve()调用来替换system()调用。exec函数系列不使用完整的shell解释器，所以不容易遭受命令行注入攻击，诸如不合规的代码示例中所展示的情况。<br>
如果指定的文件名称不包含正斜杠字符（/），execlp()，execvp()，以及（非标准的）execvP()函数会按照与shell完全相同的操作方式来搜索可行性文件。因此，正如in ENV03-C. Sanitize the environment when invoking external programs中所描述，只有在PATH环境变量被设置为一个安全的值时，它们才能在没有正斜杠字符（/）的情况下使用。<br>
execl()，execle()，execv()，以及execve()函数不会执行路径名称替换操作。<br>
此外，需要小心的是，确保外部可执行程序不会被非信任用户修改，比如说，确保可执行程序对于此用户来说没有写入权限。

```
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <errno.h>
#include <stdlib.h>

void func(char *input) {
    pid_t pid;
    int status;
    pid_t ret;
    char *const args[3] = {"any_exe", input, NULL};
    char **env;

    extern char **environ;

    /* ... Sanitize arguments ... */
    pid = fork();
    if (pid == -1) {
        /* Handle error */
    } else if (pid != 0) {
        while ((ret = waitpid(pid, &status, 0)) == -1) {
            if (errno != EINTR) {
                /* Handle error */
                break;
            }
        }
        if ((ret != -1) &&
            (!WIFEXITED(status) || !WEXITSTATUS(status)) ) {
                /* Report unexpected child status */
        }
    } else {
        /* ... Initialize env as a sanitized copy of environ ... */
        if (execve("/usr/bin/any_cmd", args, env) == -1) {
            /* Handle error */
            _Exit(127);
        }
    }
}
```
此合规的解决方案明显不同于前面的不合规代码示例。首先，input被合并到args数组中并作为一个参数传递给execve()，就不需要考虑在格式化命令字符串时遇到的缓冲区溢出或者字符串截断的问题了。其次，该合规解决方案先创建了一个新的进程，然后在子进程中执行"/usr/bin/any_cmd"。尽管这种方法比调用system()要复杂很多，但为了获得加强的安全性，付出的这些额外的工作是值得的。<br>
在未找到命令时，shell会将退出状态的值设置为127，并且POSIX建议应用程序也应该采用同样的做法。XCU, Section 2.8.2, of Standard for Information
Technology—Portable Operating System Interface (POSIX®), Base Specifications, Issue 7 [IEEE
Std 1003.1:2013],指出
>如果命令没有找到，退出状态应该为127。如果命令名称找到了，但不是一个可执行的实用程序，退出状态应该为126。使用shell来调用实用程序的应用程序应该使用这些退出状态值来报告类似的错误。

#### 11.4.3 合规的解决方案（Windows）
本合规的解决方案采用Microsoft Windows的CreateProcess() API:
```
#include <Windows.h>

void func(TCHAR *input) {
    STARTUPINFO si = { 0 };
    PROCESS_INFORMATION pi;

    si.cb = sizeof(si);
    if (!CreateProcess(TEXT("any_cmd.exe"), input, NULL, NULL, FALSE,
                       0, 0, 0, &si, &pi)) {
        /* Handle error */
    }

    CloseHandle(pi.hThread);
    CloseHandle(pi.hProcess);
}
```
此合规的解决方案依赖于input参数为非const值。如果input为const，该解决方案就需要创建此参数的一个拷贝，因为CreateProcess()函数可能会修改传递给新创建进程的命令行参数。<br>
本解决方案创建的进程，子进程不会继承父进程的任何句柄，遵循 WIN03-C. Understand HANDLE inheritance。

#### 11.4.4 不合规的代码示例（POSIX）
此不合规的代码调用C语言system()函数来删除用户主文件夹中的.config文件。
```
#include <stdlib.h>

void func(void) {
    system("rm ~/.config");
}
```
如果此易受攻击的程序具有提升的执行权限，则攻击者就可以通过操纵HOME环境变量的值来删除系统中任何位置的任何一个名为.config的文件。

#### 11.4.5 合规的解决方案（POSIX）
调用system()来执行外部程序进行必须的操作的一种替代方式是，使用现有的库接口调用在程序中直接实现其功能。此合规的解决方案调用POSIX unlink()函数删除一个文件，没有调用system()函数[IEEE Std 1003.1:2013]：
```
#include <pwd.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>

void func(void) {
    const char *file_format = "%s/.config";
    size_t len;
    char *pathname;
    struct passwd *pwd;

    /* Get /etc/passwd entry for current user */
    pwd = getpwuid(getuid());
    if (pwd == NULL) {
        /* Handle error */
    }

    /* Build full path name home dir from pw entry */

    len = strlen(pwd->pw_dir) + strlen(file_format) + 1;
    pathname = (char *)malloc(len);
    if (NULL == pathname) {
        /* Handle error */
    }
    int r = snprintf(pathname, len, file_format, pwd->pw_dir);
    if (r < 0 || r >= len) {
        /* Handle error */
    }
    if (unlink(pathname) != 0) {
        /* Handle error */
    }

    free(pathname);
}
```
在pathname(文件名称)的最后一部分为符号链接的情况下，unlink()函数也不容易遭受到符号链接攻击，因为unlink()会删除符号链接，同时不会影响任何以符号链接内容命名的文件或者目录。（参见 FIO01-C. Be careful using functions that use file names for identification。）虽然unlink()函数降低了受攻击的可能性，但无法完全避免。如果包含在pathname中的一个目录名称为符号链接，unlink()函数仍然可能遭受到攻击。这种情况可能会导致unlink()函数删除另外一个文件夹中的同名文件。

#### 11.4.6 合规的解决方案（Windows）
此合规的解决方案采用Microsoft Windows的 SHGetKnownFolderPath() API来获取当前用户的“我的文档”文件夹，然后将其与文件名组合以产生待删除文件的路径。然后使用DeleteFile() API来删除该文件。
```
#include <Windows.h>
#include <ShlObj.h>
#include <Shlwapi.h>

#if defined(_MSC_VER)
#pragma comment(lib, "Shlwapi")
#endif

void func(void) {
    HRESULT hr;
    LPWSTR path = 0;
    WCHAR full_path[MAX_PATH];

    hr = SHGetKnownFolderPath(&FOLDERID_Documents, 0, NULL, &path);
    if (FAILED(hr)) {
        /* Handle error */
    }
    if (!PathCombineW(full_path, path, L".config")) {
        /* Handle error */
    }
    CoTaskMemFree(path);
    if (!DeleteFileW(full_path)) {
        /* Handle error */
    }
}
```

#### 11.4.7 例外情况
ENV33-C-EX1：允许使用一个空指针参数来调用system()以判断用于该系统的命令处理程序是否存在。

#### 11.4.8 风险评估


### 11.5 ENV34-C. 不要保存某些函数返回的指针
C语言标准，7.22.4.6小节，第4段 [ISO/IEC 9899:2011]规定：
>getenv函数返回一个指向字符串的指针，该字符串与匹配的列表成员相关。程序不要修改所指向的字符串，该字符串可能会被后续的getenv函数调用所覆盖。

此段规定给具体实现提供了余地，比如说，可以返回一个指向静态分配的缓冲区的指针。因此，不要保存此指针，因为它指向的字符串数据可能会被下一次调用getenv()函数所覆盖，或者会由于修改了环境变量而失效。该字符串应该立即引用然后丢弃。如果预料到后面还会使用到该字符串，应该就其拷贝下来，使得在需要的时候可以安全的应用该拷贝。<br>
getenv()函数不是线程安全的。务必处理好使用此函数所带来的任何竞争条件。<br>
asctime()，localconv()，setlocale()，以及strerror()函数具有类似的限制。在后续的调用之后，就不要再访问前一次调用所返回的对象了。

#### 11.5.1 不合规的代码示例
此不合规的代码示例试图比较TMP和TEMP环境变量的值是否相同：
```
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

void func(void) {
    char *tmpvar;
    char *tempvar;
    tmpvar = getenv("TMP");
    if (!tmpvar) {
        /* Handle error */
    }
    tempvar = getenv("TEMP");
    if (!tempvar) {
        /* Handle error */
    }
    if (strcmp(tmpvar, tempvar) == 0) {
        printf("TMP and TEMP are the same.\n");
    } else {
        printf("TMP and TEMP are NOT the same.\n");
    }
}
```
此代码示例不合规是因为tmpvar所引用的字符串可能会被覆盖为第二次调用getenv()函数所得到的结果。因此，即使这两个环境变量的值不同，但tmpvar和tempvar比较得到的结果仍然可能是相等的。<br>

#### 11.5.2 合规的解决方案
此合规的解决方案使用malloc()和strcpy()函数将getenv()返回的字符串拷贝到一个动态分配的缓冲区中：
```
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

void func(void) {
    char *tmpvar;
    char *tempvar;

    const char *temp = getenv("TMP");
    if (temp != NULL) {
        tmpvar = (char *)malloc(strlen(temp)+1);
        if (tmpvar != NULL) {
            strcpy(tmpvar, temp);
        } else {
            /* Handle error */
        }
    } else {
        /* Handle error */
    }

    temp = getenv("TEMP");
    if (temp != NULL) {
        tempvar = (char *)malloc(strlen(temp)+1);
        if (tempvar != NULL) {
            strcpy(tempvar, temp);
        } else {
            /* Handle error */
        }
    } else {
        /* Handle error */
    }

    if (strcmp(tmpvar, tempvar) == 0) {
        printf("TMP and TEMP are the same.\n");
    } else {
        printf("TMP and TEMP are NOT the same.\n");
    }

    free(tmpvar);
    free(tempvar);
}
```

#### 11.5.3 合规的解决方案（Annex K）
C语言标准，Annex K，提供了getenv_s()函数用于从当前的环境中获取一个值。然而，getenv_s()函数仍然可能会与修改环境变量列表的其他线程执行存在着数据竞争。
```
#define __STDC_WANT_LIB_EXT1__ 1
#include <errno.h>
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

void func(void) {
    char *tmpvar;
    char *tempvar;
    size_t requiredSize;
    errno_t err;

    err = getenv_s(&requiredSize, NULL, 0, "TMP");
    if (err) {
        /* Handle error */
    }
    tmpvar = (char *)malloc(requiredSize);
    if (!tmpvar) {
        /* Handle error */
    }

    err = getenv_s(&requiredSize, tmpvar, requiredSize, "TMP" );
    if (err) {
        /* Handle error */
    }

    err = getenv_s(&requiredSize, NULL, 0, "TEMP");
    if (err) {
        /* Handle error */
    }
    tempvar = (char *)malloc(requiredSize);
    if (!tempvar) {
        /* Handle error */
    }
    err = getenv_s(&requiredSize, tempvar, requiredSize, "TEMP" );
    if (err) {
        /* Handle error */
    }

    if (strcmp(tmpvar, tempvar) == 0) {
        printf("TMP and TEMP are the same.\n");
    } else {
        printf("TMP and TEMP are NOT the same.\n");
    }

    free(tmpvar);
    tmpvar = NULL;
    free(tempvar);
    tempvar = NULL;
}
```

#### 11.5.4 合规的解决方案（Windows）
Microsoft Windows提供了_dupenv_s()和wdupenv_s()函数用于从当前的环境变量中获取某一个值【MSDN】。_dupenv_s()函数用指定的环境变量名称查找环境变量列表。如果找到了该名称，则分配一个缓冲区，将变量的值拷贝到缓冲区中，然后将缓冲区的地址和元素的个数返回。_dupenv_s()和_wdupenv_s()函数为getenv_s()和_wgetenv_s()提供了更加方便的替代手段，因为他们直接处理了缓冲区的分配。<br>
调用者负责调用free()，来释放由这些函数所返回的任何已分配缓冲区。
```
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#include <stdio.h>

void func(void) {
    char *tmpvar;
    char *tempvar;
    size_t len;

    errno_t err = _dupenv_s(&tmpvar, &len, "TMP");
    if (err) {
        /* Handle error */
    }

    err = _dupenv_s(&tempvar, &len, "TEMP");
    if (err) {
        /* Handle error */
    }

    if (strcmp(tmpvar, tempvar) == 0) {
        printf("TMP and TEMP are the same.\n");
    } else {
        printf("TMP and TEMP are NOT the same.\n");
    }

    free(tmpvar);
    tmpvar = NULL;
    free(tempvar);
    tempvar = NULL;
}
```

#### 11.5.5 合规的解决方案（POSIX）
POSIX提供了strdup()函数，该函数可以创建环境变量字符串的一份拷贝 [IEEE Std 1003.1:2013]。《Extensions to the C Library—Part II 》 [ISO/IEC TR 24731-2:2010]中也同样包含了strdup()函数。
```
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

void func(void) {
    char *tmpvar;
    char *tempvar;

    const char *temp = getenv("TMP");
    if (temp != NULL) {
        tmpvar = strdup(temp);
        if (tmpvar == NULL) {
            /* Handle error */
        }
    } else {
        /* Handle error */
    }

    temp = getenv("TEMP");
    if (temp != NULL) {
        tempvar = strdup(temp);
        if (tempvar == NULL) {
            /* Handle error */
        }
    } else {
        /* Handle error */
    }

    if (strcmp(tmpvar, tempvar) == 0) {
        printf("TMP and TEMP are the same.\n");
    } else {
        printf("TMP and TEMP are NOT the same.\n");
    }

    free(tmpvar);
    tmpvar = NULL;
    free(tempvar);
    tempvar = NULL;
}
```

#### 11.5.6 风险评估
保存getenv()，loacalenv()，setlocale()，或strerror()返回的字符串指针可能会导致数据被覆盖。

|规则|严重性|可能性|修补成本|优先级|级别|
|---|---|---|---|---|---|
|ENV34-C|低|可能|中|P4|L3|

