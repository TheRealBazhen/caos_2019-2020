


# Динамические библиотеки


<br>
<div style="text-align: right"> Спасибо ??? за участие в написании текста </div>
<br>


Сегодня в программе:
* Создание и подгрузка динамической библиотеки
  * <a href="#create_dynlib" style="color:#856024">Создание</a>
  * Подгрузка 
    1. <a href="#load_с" style="color:#856024">При старте средствами OS (динамическая компоновка)</a> 
    <br> Вот [это](https://www.ibm.com/developerworks/ru/library/l-dynamic-libraries/) можно почитать для понимания, что в этом случае происходит.
    2. В произвольный момент времени:
      * <a href="#load_python" style="color:#856024">Из python</a> 
      * <a href="#load_с_std" style="color:#856024">Из программы на С (dlopen)</a> 
      * <a href="#load_с_mmap" style="color:#856024">Из программы на С с извращениями (mmap)</a> 
* Нетривиальный пример применения динамических библиотек
  <br> <a href="#c_interpreter" style="color:#856024">Развлекаемся и пишем простенький интерпретатор языка C (с поблочным выполнением команд)</a> 
  
  
<a href="#hw" style="color:#856024">Комментарии к ДЗ</a>

[Ридинг Яковлева](https://github.com/victor-yacovlev/mipt-diht-caos/tree/master/practice/function-pointers)

# <a name="create_dynlib"></a> Создание динамической библиотеки 


```cpp
%%cpp lib.c
%# `-shared` - make shared library
%# `-fPIC` - make Positional Independant Code
%run gcc -Wall -shared -fPIC lib.c -o libsum.so # compile shared library
%run objdump -t libsum.so | grep sum # symbols in shared library filtered by 'sum'

int sum(int a, int b) {
    return a + b;
}

float sum_f(float a, float b) {
    return a + b;
}
```


Run: `gcc -Wall -shared -fPIC lib.c -o libsum.so # compile shared library`



Run: `objdump -t libsum.so | grep sum # symbols in shared library filtered by 'sum'`


    libsum.so:     file format elf64-x86-64
    0000000000000634 g     F .text	000000000000001a sum_f
    0000000000000620 g     F .text	0000000000000014 sum


# <a name="load_python"></a> Загрузка динамической библиотеки из python'а


```python
import ctypes

lib = ctypes.CDLL("./libsum.so")
%p lib.sum(3, 4) # По умолчанию считает типы int'ами, поэтому в этом случае все хорошо
%p lib.sum_f(3, 4) # А здесь python передает в функцию инты, а она принимает float'ы. Тут может нарушаться соглашение о вызовах и происходить что угодно

# Скажем, какие на самом деле типы в сигнатуре функции
lib.sum_f.argtypes = [ctypes.c_float, ctypes.c_float]
lib.sum_f.restype = ctypes.c_float
%p lib.sum_f(3, 4) # Теперь все работает хорошо
```


`lib.sum(3, 4) = 7`  # По умолчанию считает типы int'ами, поэтому в этом случае все хорошо



`lib.sum_f(3, 4) = 0`  # А здесь python передает в функцию инты, а она принимает float'ы. Тут может нарушаться соглашение о вызовах и происходить что угодно



`lib.sum_f(3, 4) = 7.0`  # Теперь все работает хорошо


# <a name="load_с"></a> Загрузка динамической библиотеки из программы на С. Стандартными средствами, автоматически при старте программы


```cpp
%%cpp ld_exec_dynlib_func.c
%# `-lsum` - подключаем динамическую библиотеку `libsum.so`
%# `-L.` - во время компиляции ищем библиотеку в директории `.`
%# `-Wl,-rpath -Wl,'$ORIGIN/'.` - говорим линкеру, чтобы он собрал программу так
%# чтобы при запуске она искала библиотеку в `'$ORIGIN/'.`. То есть около исполняемого файла программы
%run gcc -Wall -g ld_exec_dynlib_func.c -L. -lsum -Wl,-rpath -Wl,'$ORIGIN/'. -o ld_exec_dynlib_func.exe
%run ./ld_exec_dynlib_func.exe

#include <stdio.h>

// объявляем функции
int sum(int a, int b);
float sum_f(float a, float b);

int main() {  
    #define p(stmt, fmt) printf(#stmt " = " fmt "\n", stmt);
    p(sum(1, 1), "%d");
    p(sum(40, 5000), "%d");
    
    p(sum_f(1, 1), "%0.2f");
    p(sum_f(4.0, 500.1), "%0.2f");
    return 0;
}
```


Run: `gcc -Wall -g ld_exec_dynlib_func.c -L. -lsum -Wl,-rpath -Wl,'$ORIGIN/'. -o ld_exec_dynlib_func.exe`



Run: `./ld_exec_dynlib_func.exe`


    sum(1, 1) = 2
    sum(40, 5000) = 5040
    sum_f(1, 1) = 2.00
    sum_f(4.0, 500.1) = 504.10


# <a name="load_с_std"></a> Загрузка динамической библиотеки из программы на С в произвольный момент времени, используя стандартные функции


```cpp
%%cpp stdload_exec_dynlib_func.c
%# `-ldl` - пародоксально, но для подгрузки динамических библиотек, нужно подгрузить динамическую библиотеку
%run gcc -Wall -g stdload_exec_dynlib_func.c -ldl -o stdload_exec_dynlib_func.exe
%run ./stdload_exec_dynlib_func.exe

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/mman.h>
#include <fcntl.h>
#include <assert.h>
#include <dlfcn.h>

// объявляем функции
int sum(int a, int b);
float sum_f(float a, float b);

int main() {  
    
    void *lib_handle = dlopen("./libsum.so", RTLD_NOW);
    if (!lib_handle) {
        fprintf(stderr, "dlopen: %s\n", dlerror());
        abort();
    }
   
    int (*sum)(int, int) = dlsym(lib_handle, "sum");
    float (*sum_f)(float, float) = dlsym(lib_handle, "sum_f");
    
    #define p(stmt, fmt) printf(#stmt " = " fmt "\n", stmt);
    p(sum(1, 1), "%d");
    p(sum(40, 5000), "%d");
    
    p(sum_f(1, 1), "%0.2f");
    p(sum_f(4.0, 500.1), "%0.2f");
    
    dlclose(lib_handle);
    return 0;
}
```


Run: `gcc -Wall -g stdload_exec_dynlib_func.c -ldl -o stdload_exec_dynlib_func.exe`



Run: `./stdload_exec_dynlib_func.exe`


    sum(1, 1) = 2
    sum(40, 5000) = 5040
    sum_f(1, 1) = 2.00
    sum_f(4.0, 500.1) = 504.10


# <a name="load_с_std"></a> Загрузка динамической библиотеки из программы на С в произвольный момент времени, используя mmap

В примере отсутствует парсинг elf файла, чтобы выцепить адреса функций. Поэтому они просто захардкожены


```cpp
%%cpp mmap_exec_dynlib_func.c
%run gcc -Wall -fsanitize=address -g mmap_exec_dynlib_func.c -o mmap_exec_dynlib_func.exe
%run ./mmap_exec_dynlib_func.exe

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/mman.h>
#include <fcntl.h>
#include <assert.h>

int main() {
    int fd = open("libsum.so", O_RDWR);
    struct stat s;
    assert(fstat(fd, &s) == 0);
    void* mapped = mmap(
        /* desired addr, addr = */ NULL, 
        /* length = */ s.st_size, 
        /* access attributes, prot = */ PROT_READ | PROT_EXEC | PROT_WRITE, // обратите внимание на PROT_EXEC
        /* flags = */ MAP_SHARED,
        /* fd = */ fd,
        /* offset in file, offset = */ 0
    );
    assert(close(fd) == 0); // Не забываем закрывать файл (при закрытии регион памяти остается доступным)
    if (mapped == MAP_FAILED) {
        perror("Can't mmap");
        return -1;
    }
 
    int (*sum)(int, int) = (void*)((char*)mapped + 0x620); // 0x620 - тот самый оффсет из objdump'a
    float (*sum_f)(float, float) = (void*)((char*)mapped + 0x634); 
    
    #define p(stmt, fmt) printf(#stmt " = " fmt "\n", stmt);
    
    p(sum(1, 1), "%d");
    p(sum(40, 5000), "%d");
    
    p(sum_f(1, 1), "%0.2f");
    p(sum_f(4.0, 500.1), "%0.2f");

    assert(munmap(
        /* mapped addr, addr = */ mapped, 
        /* length = */ s.st_size
    ) == 0);
    return 0;
}
```


Run: `gcc -Wall -fsanitize=address -g mmap_exec_dynlib_func.c -o mmap_exec_dynlib_func.exe`



Run: `./mmap_exec_dynlib_func.exe`


    sum(1, 1) = 2
    sum(40, 5000) = 5040
    sum_f(1, 1) = 2.00
    sum_f(4.0, 500.1) = 504.10


# <a name="c_interpreter"></a> Простенький интерпретатор для С

Идея такая: на каждый кусочек кода будем компилировать динамическую библиотеку, подгружать ее, и выполнять из нее функцию, в которой будет этот самый кусочек.

Взаимодействие между кусочками через глобальные переменные. (Все кусочки кода выполняются в основном процессе.)

Каждая динамическя библиотека компонуется со всеми предыдущими, чтобы видеть их глобальные переменные. Для этого же при загрузке библиотек берется опция RTLD_GLOBAL.


```python
!rm -r tmp a.txt
!mkdir tmp
```


```python
import os
import subprocess
import ctypes
from textwrap import dedent

uniq_counter = globals().get("uniq_counter", 0) + 1
libs = []
all_includes = []
all_variables = []


def add_includes_c(includes):
    global all_includes
    all_includes = list(set(all_includes) | set(includes.split('\n')))

    
def declare_c(declaration):
    assignment_pos = declaration.find('=')
    assignment_pos = assignment_pos if assignment_pos != -1 else len(declaration)
    decl_part = declaration[:assignment_pos].rstrip()
    var_name_begin = decl_part.rfind(' ')
    var_assignment = declaration[var_name_begin:]
    interprete_c(var_assignment, variables=[decl_part])

    
def interprete_c(code="", variables=[]):
    func_name = "lib_func_%d_%d" % (uniq_counter, len(libs))
    source_name = "./tmp/" + func_name + ".c"
    lib_name = "lib" + func_name + ".so"
    lib_file = "./tmp/" + lib_name
    includes_list = "\n".join(all_includes)
    variables_list = "; ".join("extern " + v for v in all_variables) + "; " + "; ".join(variables)
    out_file = "./tmp/" + func_name + ".out" 
    err_file = "./tmp/" + func_name + ".err" 
    lib_code = dedent('''\
        #include <stdio.h>
        {includes_list}
        {variables_list};
        void {func_name}() {{
            freopen("{err_file}", "w", stderr);
            freopen("{out_file}", "w", stdout);
            {code};
            fflush(stderr);
            fflush(stdout);
        }}
        ''').format(**locals())
    with open(source_name, "w") as f:
        f.write(lib_code)
    compile_cmd = (
        ["gcc", "-Wall", "-shared", "-fPIC", source_name, "-Ltmp"] + 
        ['-l' + lib_f for lib_f in libs] + 
        ["-Wl,-rpath", "-Wl," + os.path.join(os.getcwd(), "tmp"), "-o", lib_file]
    )
    try:
        subprocess.check_output(compile_cmd)
    except:
        print("%s\n%s" % (lib_code, " ".join(compile_cmd)))
        get_ipython().run_cell("!" + " ".join(compile_cmd))
        raise
    
    lib = ctypes.CDLL(lib_file, ctypes.RTLD_GLOBAL)  # RTLD_GLOBAL - важно! Чтобы позднее загруженные либы видели ранее загруженные
    func = lib[func_name]
    func()
    for fname, stream in [(err_file, sys.stderr), (out_file, sys.stdout)]:
        with open(fname, "r") as f:
            txt = f.read()
            if txt:
                print(txt, file=stream)
    libs.append(func_name)
    all_variables.extend(variables)
    
```


```python
interprete_c(r'''
    printf("%d", 40 + 2); 
    dprintf(2, "Hello world!");
''')
```

    42


    Hello world!



```python
add_includes_c('''
    #include <math.h>"
''')
interprete_c('''
    printf("%f", cos(60.0 / 180 * 3.1415))
''')
```

    0.500027



```python
declare_c('''
   int a = 4242
''')
```


```python
interprete_c('''
    printf("1) %d", a);
''')
interprete_c('''
    printf("2) %06d", a);
''')
interprete_c('''
    printf("3) %6d", a);
''')
interprete_c('''
    printf("4) %0.2f", (float)a);
''')
```

    1) 4242
    2) 004242
    3)   4242
    4) 4242.00



```python
add_includes_c('''
    #include <sys/types.h>
    #include <sys/stat.h>
    #include <fcntl.h>
    #include <unistd.h>
''')
declare_c('''
    int fd = open("./a.txt", O_WRONLY | O_CREAT, 0644)
''')
interprete_c('''
    dprintf(fd, "Hello students! a = %d", a);
    close(fd);
    printf("a.txt written and closed!");
''')
```

    a.txt written and closed!



```python
!cat a.txt
```

    Hello students! a = 4242


```python

```


```python

```


```python

```

# <a name="hw"></a> Комментарии к ДЗ

* 


```python

```


```python

```
