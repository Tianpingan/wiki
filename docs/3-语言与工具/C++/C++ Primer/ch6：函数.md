## 6.1 函数基础
一个典型的函数包括：
- 返回类型
- 函数名称
- 由0个或多个形参组成的列表
- 函数体

通过**调用运算符**（一对圆括号）来执行函数。
调用运算符作用于一个表达式，该表达式是函数或者指向函数的指针。
圆括号之内是用括号隔开的**实参**列表，我们用实参列表初始化函数的形参（初始化的顺序不一定按列表顺序），调用表达式的类型就是函数的返回类型。

> 实参和形参的区别的什么？
> 实参是函数调用的实际值，是形参的初始值。

### 6.1.1 局部变量
名字有**作用域**，对象有**生存周期**。

- 名字的作用域是程序文本的一部分，名字在其中**可见**
- 对象的生命周期是程序执行过程中该对象存在的一段时间。
> 
> 变量和对象的关系：
> 通常情况下，对象是指一块能存储数据并具有某种内存的内存空间
> 一些人仅在与类相关的情景下才使用“对象”这个词。另一些人则把已命名的对象和未命名的对象区分开来，他们把已命名的对象叫做变量。还有一些人把对象和值区分开来，其中对象是能被程序修改的数据，而值指只读的数据。

