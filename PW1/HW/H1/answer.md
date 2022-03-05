**问题1-1**

`-D` 选项用来在 `gcc` 编译时定义宏，在命令中添加 `-DNEG` 选项后定义了 `NEG`宏，生成的 `sample.i` 文件中 `M` 被定义为 -4

**问题1-2**

`sample-32.s` 和 `sample.s` 核心汇编代码的区别以及产生区别的原因：

* `pushl` 和 `pushq` : `pushl %epb` 表示将4 Byte长度的数据压入栈( `epb` 是32位环境下的栈基址寄存器)，`pushq %rbq` 则是8 Byte(`rbq` 是64位环境下的栈基址寄存器) 
* `esp` 和 `rsp` : `esp` 是32位环境下的栈顶地址寄存器，`rsp` 是64位环境下的栈顶地址寄存器
* `sample-32.s` 中的 `subl` : 64位环境下 `rsp` 指向位置(栈顶)以外的128字节空间是保留给当前调用的函数使用的，所以不需要额外分配栈空间，但是32位环境下就需要使用 `subl` 指令改变 `esp` 指向的位置来分配空间
* `leave` 和 `popq` : `leave` 相当于 `popl %ebp ` ，这样就和 `popq %rbq` 相对应
