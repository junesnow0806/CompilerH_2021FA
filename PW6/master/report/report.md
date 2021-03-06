# PW6 实验报告

PB19061283 黄科鑫 PB19051175 梁峻滔 PB19051001 张一方

## 问题回答

### 1-1

答：clang 给出的`while_test.sy` 的有关while的LLVM IR如下：

```LLVM IR
...
  br label %2

2:                                                ; preds = %5, %0
  %3 = load i32, i32* @a, align 4
  %4 = icmp sgt i32 %3, 0
  br i1 %4, label %5, label %11

5:                                                ; preds = %2
  %6 = load i32, i32* @b, align 4
  %7 = load i32, i32* @a, align 4
  %8 = add nsw i32 %6, %7
  store i32 %8, i32* @b, align 4
  %9 = load i32, i32* @a, align 4
  %10 = sub nsw i32 %9, 1
  store i32 %10, i32* @a, align 4
  br label %2

11:                                               ; preds = %2
...
```

代码布局的特点是：条件判断（`%2`)和语句块(`%5`)分为两个基本块，在while语句之后创建条件不成立时的基本块（`%11`）。在进入while语句时，首先无条件跳转（如上述代码中label 2之前的`br label %2`）到条件判断基本块；在条件判断基本块的末尾进行条件跳转（如上述代码中的`br i1 %4, label %5, label %11`）到语句块或者while语句之后的第一个基本块（`%11`）；在语句块的末尾无条件跳转到条件判断基本块（如上述代码中label 11之前的`br label %2`）。

### 1-2

函数调用语句对应的 LLVM IR 代码特点：

* `%8 = call i32 @add(i32 %5, i32 %6)`

  由`call`表示函数调用，第一个参数为返回值类型，第二个参数为函数名，紧跟函数名后面是形参列表，每一项形参包括类型和值(即中间变量)。

* `define dso_local i32 @add(i32, i32) #0 {...}`

  定义一个函数时参数可以只声明类型而不必声明变量名，函数体中 %0, #1, ...依次是第一个、第二个...参数的值。

### 2-1

两种`getelementptr`方法：

- `%2 = getelementptr [10 x i32], [10 x i32]* %1, i32 0, i32 %0`
- `%2 = getelementptr i32, i32* %1, i32 %0`

第一种用来获取数组元素所在地址的，第二种用来获取指针的地址。

- 对于第一种用法，第一个偏移0首先计算得到%1所指向的数组的首元素，第二个偏移0计算得到首元素向后偏移0个i32的元素，即其自身，然后该指令返回计算所得元素的地址。

- 对于第二种用法，第一个偏移0计算得到%1所指的i32变量，然后返回该变量的地址

我认为`gep`指令需要第一个%0是为了精确计算得到变量的宽度，在`Instruction.cpp`的`Type *GetElementPtrInst::get_element_type`方法中也可以看到，其返回的被设置到`element_ty_`中的值，实际上是指针所指的变量类型，而非指针本身类型，这样做就是要方便地址的计算。

### 3-1

 - 允许变量和函数重名
 - 分开存储分开查找的效率更高

## 实验设计

- **全局变量初始化** 在全局变量的初始化中，可能出现立即数、const 变量值组成的	表达式，通过`scope.in_global()`可以为每个表达式作特殊处理，但这样会使得`IRBuilder.cpp`的代码过于臃肿，因此我们通过一个全局值栈来封装了底层的一些操作，并通过提供简单的几个接口：入栈、出栈、根据op计算栈顶值、添加const常量等，就可以简单地实现在检测到全局变量时对常量表达式地计算。 

	具体来说，就是在`GlobalVal`类中，维护一个`flag_`栈（用于记录栈中对应位置应该为INT还是FLOAT类型）；常量值栈`iVal_`,`fVal_`，用于保存计算出的值和立即数；const变量栈`cInt_`,`cFloat_`，用于保存const变量值。

- **数组作函数形参** 数组作为函数形参时，由于不知道其长度，所以需要把它当作指针处理。而这个时候，就会引发指针类型作实参和形参不匹配的问题，解决方法是：数组做实参时，就取其第一个元素的地址作为参数（只考虑一维数组），而函数内的指针类型递归调用的时候，即指针作实参，就应该直接`load`.

- **函数返回值** 在函数声明的时候就为返回值分配空间并声明一个基本块，用来处理返回值，此后处理到的每一个`return`语句，都通过设置预先分配好的返回值并`br`到基本块中来实现。（对于没有显式`return`语句的函数，可以在`FuncDef`函数末尾通过`builder->get_insert_block()->get_terminator()`判断）

- **数组变量的长度** 数组变量声明时，需要获取数组的长度，这个值同样可以是立即数、const 变量值组成的	表达式，通过设置一个`_get_literal`标志，再次利用全局变量初始化的栈

- **局部变量初始化** 由于局部变量的值不确定，所以使用`load/store`即可

- **左值表达式** 虽然命名为“左值”，但是事实上也可以作为表达式（例如`a=b+c`中`a,b,c`都属于左值表达式），因此在左值表达式的处理中，既要计算地址值，又要计算实际的值。

- **表达式的处理** 用`switch`对不同操作符分开处理，使用全局变量`_exp_val`存放已经分析完的表达式，左/右操作数(本身也是表达式)分析完(调用`accept`)后从`_exp_val`中取出用于分析当前表达式，当前表达式分析完后也存到`_exp_val`中，分析表达式过程中根据操作数的类型，进行必要的类型转换，然后调用对应的生成指令API。

- **短路运算** 

- **分支与循环的处理**

## 实验难点及解决方案

- **类型转换** 一开始考虑到了太多可能出现的各种类型，例如在表达式中出现 INT1 + INT32、指针运算等，导致类型转换的部分写得比较复杂。后来理清了其中的关系，很快就写完了

- **return语句** 刚开始的实现是在哪里return就在哪里创建ret语句，但是这样容易带来block内的多个结束指令，最终参考了Clang生成的代码，采用统一ret的方式就解决了问题

- **全局变量初始化** 一开始没有考虑literal和const组成的表达式，且实现的方式很原始，就是使用一堆flag和条件判断，后来将逻辑理清，采用一个类封装好并利用类型重载，就简洁而明了

- **数组、指针的使用** 这里主要是牵扯到数组做参数的时候引发的一系列类型、寻址的问题。在仔细阅读文档并测试后，就明白如何正确使用相关类型

- **函数参数** 一开始不知道要用函数类型自带的args，后来通过助教的代码示例解决了。

- **其他问题** ……

## 实验总结

通过本次实验完整地感受了中间代码的生成过程。本次实验需要我们对编译器的语法、语义分析有清晰的认知，需深入理解助教所设计的接口，并熟练掌握C++编程，对我们来说是很大的挑战，同样也是一种成长。 起初看到空白的22个函数一头雾水，不过在我们组仔细阅读了文档之对实验要求有了大致了解后，大家深入探讨和交流，于是也大致明白应该如何构造我们的函数了。我们按照函数变量声明、表达式、控制流的方式分配任务，每写一个函数都加深了我们对编译的认知。 最后DEBUG的时候，虽然遇到了很多问题，特别是函数变量定义、选择和循环、还有类型转换上进行了不断的修改和尝试，最终本地测试的问题迎刃而解。

## 实验反馈

希望不要有隐藏样例了。。

## 组间交流

无

> 注：测试样例的设计参考了编译大赛中提供的测试集