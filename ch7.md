# ch7 链接  
## 〇. 本章中的概念
> 1. **全局变量**: 特指 `nonstatic` 的全局变量
> 3. **局部变量**: 特指 `nonstatic` 的局部变量
> 2. **静态变量**: 所有被 `static` 修饰的变量
## 一. 编译器驱动程序
> 从`main.c`源代码到`prog`可执行文件包含以下几个步骤
>- 其中`sum.o`中的某些符号被`main.c`所引用
1. 预处理
```sh
cpp main.c main.i
```
2. 编译
```sh
cc1 main.i -Og -o main.s
```
3. 汇编
```sh
as -o main.o main.s
```
4. 链接
```sh
ld -o prog main.o sum.o
```
5. 执行
```sh
./prog
```
## 二.静态链接
上述 `ld` 命令就是 `linux` 的静态链接器
> 链接器需要完成两个任务
>> 1. 符号解析(*symbol resolution*): 目标文件(*object file*)**定义**和**引用**符号.每个符号对应一个函数,一个全局变量或一个静态(*static*)变量.符号解析的目的是将每个**符号引用**和对应的**符号定义**关联起来.
>> 2. 重定位(*relocation*): **编译器**和**汇编器**生成从地址0开始的代码和数据**节**.链接器通过将每个**符号定义**和一个**内存位置**关联起来,然后修改所有对这些符号的引用,使得它们指向这个内存位置,从而重定位这些节,.链接器通过**汇编器**产生的**重定位条目**进行重定位.
## 三.目标文件(*object file/module*)
### 1. 目标文件的分类(三种)
>- 可重定位目标文件(编译器和汇编器生成): 包含二进制代码和数据,可以在编译时和其他可重定位目标文件合并而得到一个可执行目标文件.
>- 可执行目标文件(链接器生成): 包含二进制代码和数据,可以直接复制到内存并执行
>- 共享目标文件(编译器和汇编器生成): 一种特殊的可重定位目标文件,可以在**加载**或**运行时**被动态加载到内存并链接.
### 2. 目标文件的格式分类(三种)
| 系统 | 目标文件格式 |
| -- | -- |
|Windows|**PE**(*Portable Executable*)|
|MacOS|**Mach-O**|
|linux,unix|**ELF**(*Executable and Linkable Format*)|  

>本章介绍 **ELF**.
### 3. ELF 可重定位目标文件格式
| section | description |
| - | - |
|ELF header|16B的序列(生成此文件的系统的字大小和字节顺序) + 协助链接器工作的信息(ELF header的大小,目标文件的类型eg.可重定位/可执行/共享,机器类型eg.X86,section header table的偏移以及其中条目的大小和数量)|
|.text|程序的机器代码|
|.rodata|只读数据 (e.g. `printf`的格式串, `switch`的跳转表)|
|.data|**已初始化**的**全局变量**和**static变量**|
|.bss|**未初始化**(或**初始化为0**)的**全局变量**和**static变量**.在目标文件中该段不占实际空间,而在运行时在内存中分配这些变量,初始化为0|
|.symtab|**符号表**,存放程序中**定义**和**引用**的**函数**,**全局变量**和**static变量**的信息.*和编译器中的符号表不同,该段不包含局部变量的条目*|
|.rel.text|记录.text中位置信息,链接时会被修改|
|.rel.data|记录所有定义和引用的全局变量的重定位信息|
|.debug|调试符号表,编译时需要`-g`选项|
|.line|原始C代码中的行号和机器指令之间的映射,编译时需要`-g`选项|
|.strtab|字符串表,包括.symtab和.debug中的符号名,section header table中的节名字.该节本质上就是一个字符串序列|
|section header table|每个节的位置和大小的条目表|
## 四. 符号和符号表
每个可重定位目标文件(记作m)都有一个符号表,包含m定义和引用的符号的信息.在链接器的视角下有三种符号
>- 由m定义, 能被其他模块引用的称为***全局符号***.对应 m 定义的 `nonstatic` 函数和全局变量
>- 由其他模块定义并被m引用的全局符号, 称为***外部符号***.对应其他模块定义的 `nonstatic` 函数和全局变量
>- 只被m定义和引用的***局部符号***.对应于 m 定义的 `static` 函数和 `static` 全局变量

>- 注意 `static` 局部变量也会在符号表中被唯一创建(如果重名则会被重命名),这种符号称为*局部链接器符号*(*local linker symbol*)

**ELF符号表**(*.symtab*)条目定义如下
```C
typedef struct { 
    int   name;      /* String table (.strtab) offset */ 
    char  type:4,    /* Function or data (4 bits) */ 
	  binding:4; /* Local symbol or global symbol (4 bits) */ 
    char  reserved;  /* Unused */  
    short section;   /* Section header index */
    long  value;     /* Section offset or absolute address */ 
    long  size;      /* Object size in bytes */ 
} Elf64_Symbol; 
```
- `section` 字段
    - section header table 的索引, 索引到一个**具体存在**的节(section)
    - 三个特殊的*伪节*(pseudosection), section header table 中**不存在**
        - ABS: 不该被重定位的符号.
        - UNDEF: 未定义的符号, 即外部符号.
        - COMMON: 未初始化的全局变量(.bss 代表未初始化的静态变量和初始化为0的全局或静态变量)

## 五. 符号解析
符号解析是指将**符号引用**与定义它的目标文件中的**符号表中**的该**符号定义**关联起来.  

符号解析分为两种:局部符号解析和外部符号解析
>- 局部符号解析
>   - 简单明了.
>   - 编译器值允许每个局部符号有一个定义.
>   - 对于静态局部变量的局部链接器符号,编译器也会保证他们有唯一的名字.
>- 全局符号解析
>   - 当编译器遇到一个不是当前模块定义的符号时,会假定它在别的模块中有定义并生成一个***链接器符号表条目***,将其交给链接器处理.
>   - 当链接器找不到这个符号的定义时,会报错并终止.

### 1. 解析多重定义的全局符号
局部符号只有定义它的模块可见,但是全局符号是所有模块可见的.如果有多个模块定义同名符号,linux下的应对策略如下:
>定义
>>**强符号**: 函数和已经初始化的全局变量
>> **弱符号**: 未初始化的全局变量
- 规则1: 不允许多个同名强符号.
- 规则2: 一个强符号和多个弱符号同名,选择那个强符号.
- 规则3: 多个弱符号同名,随机选择一个弱符号.  
### 2. 与静态链接库链接时的符号解析  
与静态链接库的链接命令如下:
```sh
gcc main.c /usr/lib/libm.a /usr/lib/libc.a
```
> 注意 `.a` 和 `.o` 文件在命令行中的顺序.
> 举例说明:如果`main.c`文件中有对外部符号的引用, 而该外部符号在`libc.a`中的`printf.o`文件中,则 `main.c` 一定要写在`libc.a`的前面,这样链接器才会保留`libc.a`中的`printf.o`.否则链接器不知道有文件对`printf.o`中的符号进行引用,进而不会保留`printf.o`文件.


## 六. 重定位
符号解析结束后,代码中的符号引用都正好和一个符号定义(一个输入模块的符号表条目)关联起来.此时就可以进行重定位了.重定位将合并代码块,为每个符号分配**运行时地址**,具体过程如下:
>- 重定位节和符号定义:将可重定位目标文件中的同名节合并,然后将运行时地址赋给新的聚合节和定义的符号
>- 重定位节中的符号引用:链接器修改代码节和数据节中对每个符号的引用,让它们指向正确的地址.

要执行重定位节中的符号引用,链接器必须依赖可重定位目标文件中的*重定位条目*.

> **重定位条目**
>- 当汇编器生成一个可重定位目标文件时,它不知道数据和代码最终存放的位置,更不知道外部定义的变量或函数的位置.因此无论何时汇编器遇到未知的外部对象的引用,它就会生成一个*重定位条目*.告诉链接器在合并可重定位目标文件时如何修改这个引用.  
>- 代码的重定位条目在`.rel.text`中,数据的重定位条目在`.rel.data`中.

重定位条目定义如下
```C
typedef struct { 
    long offset;    /* Offset of the reference to relocate */ 
    long type:32,   /* Relocation type */ 
	 symbol:32; /* Symbol table index */ 
    long addend;    /* Constant part of relocation expression */
} Elf64_Rela; 
```
- `type`字段的其中两种
    - `R_X86_64_PC32`:重定位一个使用32位PC相对寻址的引用
    - `R_X86_64_32`:重定位一个使用32位绝对地址的引用

对于一个**可重定位目标文件** (*.o*) 以上两种`type`的重定位过程
```C
for each section s {
    for each relocation entry r {
        refptr = s + r.offset /* 指向.o中要修改的引用 */

        if (r.type == R_X86_64_PC32) {
            refaddr = RUNTIMEADDR(s) + r.offset; /*运行时本可重定位目标文件的某个节的地址加上偏移量得到运行时该符号引用的地址*/
            *refptr = RUNTIMEADDR(r.symbol) - refaddr + r.addend
        }
        
        if (r.type == R_X86_64_32) {
            *refptr = RUNTIMEADDR(r.symbol) + r.addend
        }
    }
}
```