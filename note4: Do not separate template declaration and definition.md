# Do not separate template declaration and definition.
## 一:background
一般在写C++相关代码的时候，我们总是习惯于将声明与定义相分离，即分为 .h 文件与 .cpp 文件，目的是为了
1：模块化开发，将代码分为多个文件，便于多人协作
2：提高代码编译效率，修改一个文件的时候，只需要重新编译该文件，而不需要重新编译整个项目
3：提高代码复用率，通过引用头文件可以方便得引用代码

在实际的编译过程中，采用分离式编译，头文件与.cpp 文件分开编译，编译阶段不检查跨文件依赖是否正确，在链接阶段，合并所有部分

但是在模版编程中，声明与实现却是要求放在同一个文件中的，现在来探讨为什么

## 二：模版声明与定义
<span style="font-weight: bold; font-style: italic; color: red;">
C++ 编程标准：
要求编译器在实例化模版的时候，必须在上下文中看到其实现
</span>

反过来也可以这样去认为，在看到实例化模版之前，编译器对模版的实现是不处理的。原因很简单，模版参数不可知，模版类的实现是如何得知不同函数中不同变量的类型等等一系列生成符号的具体实现呢，也就不会去编译模版类的.cpp 文件

C++ 编程思想说明了原因：
由 template<>  处理的任何东西都意味着编译器在当时不会为它分配内存空间，它一直在等待着直到一个模版实例被告知。
在编译器和连接器的某一处，有一机制能去掉模版的多重定义，所以为了容易使用，几乎总是在头文件中放置全部的模板声明和定义。

简单来说：只有模板实例化时，编译器才会得知T实参是什么。而编译器在处理模板实例化时，不仅仅要看到模板的定义式，还需要模版的实现体。

为什么需要这样呢？

比如说存在类Rect, 其类定义式写在test.h，类的实现体写在test.cpp中。对于模板来说，编译器在处理test.cpp文件时，编译器无法预知T的实参是什么，所以编译器对其实现是不作处理的。

紧接着在main.cpp中用到了Rect，这个时候会实例化。也就是说，在test.h中会实例出对Rect进行类的声明。但是，由于分离式编译模式，在编译的时候只需要类的声明即可，因此编译是没有任何问题的。

但是在链接的过程中，需要找到Rect的实现部分。但是上面也说了，编译是相对于每个cpp文件而言的。在test.cpp的编译的时候，由于不知道T的实参是什么，并没有对其进行处理。因此，Rect的实现自然并没有被编译，链接也就自然而然地因找不到而出错。

例子
解释清楚了，接下来可以看一个例子：

在test.h文件中，定义模板类Rect：

```C++
#include <iostream>

template<typename T>
class Rect {
  public:
    Rect(T l = 0.0f, T t = 0.0f, T r = 0.0f, T b = 0.0f) :
      left_(l), top_(t), right_(r), bottom_(b) {}

    void display();

    T left_;
    T top_;
    T right_;
    T bottom_;
};
```

在test.cpp文件中，定义模板类Rect方法的实现：
```C++
#include "test.h"

template<typename T>
void Rect<T>::display() {
  std::cout << left_ << " " << top_ << " " << right_
    << " " << bottom_ << std::endl;
}
```

最终在main.cpp文件中，使用改模板类：
```C++
#include <iostream>
#include "test.h"

int main() {
  Rect<float> rect(1.1f, 2.2f, 3.3f, 4.4f);
  rect.display();

  return 0;
}
```

对这三个文件进行编译，最终会报错，报错的内容为：
```C++
yngzmiao@yngzmiao-virtual-machine:~/test/build$ cmake .. && make
-- The C compiler identification is GNU 4.8.4
-- The CXX compiler identification is GNU 4.8.4
-- Check for working C compiler: /usr/bin/cc
-- Check for working C compiler: /usr/bin/cc -- works
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Detecting C compile features
-- Detecting C compile features - done
-- Check for working CXX compiler: /usr/bin/c++
-- Check for working CXX compiler: /usr/bin/c++ -- works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done
-- Generating done
-- Build files have been written to: /home/yngzmiao/test/build
Scanning dependencies of target test
[ 25%] Building CXX object CMakeFiles/test.dir/test.cpp.o
[ 50%] Linking CXX static library libtest.a
[ 50%] Built target test
Scanning dependencies of target main
[ 75%] Building CXX object CMakeFiles/main.dir/main.cpp.o
[100%] Linking CXX executable main
CMakeFiles/main.dir/main.cpp.o：在函数‘main’中：
main.cpp:(.text+0x3c)：对‘Rect<float>::display()’未定义的引用
collect2: error: ld returned 1 exit status
make[2]: *** [main] 错误 1
make[1]: *** [CMakeFiles/main.dir/all] 错误 2
make: *** [all] 错误 2
```

可以看出，改代码在ld的过程中出现了错误，即链接的时候没有找到实现而出错。

如果将模板类的声明和实现不分离，都写在.h文件中。即如下：
```C++
#include <iostream>

template<typename T>
class Rect {
  public:
    Rect(T l = 0.0f, T t = 0.0f, T r = 0.0f, T b = 0.0f) :
      left_(l), top_(t), right_(r), bottom_(b) {}

    void display() {
      std::cout << left_ << " " << top_ << " " << right_
        << " " << bottom_ << std::endl;
    }

    T left_;
    T top_;
    T right_;
    T bottom_;
};
```
最终编译运行没有问题。

## 三：总结
在分离式编译的环境下，编译器编译某一个cpp文件时并不知道另一个cpp文件的存在，也不会去查找（当遇到未决符号时它会寄希望于链接器）。

这种模式在没有模板的情况下运行良好，但遇到模板时就傻眼了，因为模板仅在需要的时候才会实例化出来。所以，当编译器只看到模板的声明时，它不能实例化该模板，只能创建一个具有外部链接的符号并期待链接器能够将符号的地址决议出来。

然而当实现该模板的cpp文件中没有用到模板的实例时，编译器懒得去实例化，所以，整个工程中就找不到一行模板实例的二进制代码，于是链接器也黔驴技穷了。



