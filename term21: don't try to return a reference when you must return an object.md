# Don't try to pass a reference when you must return an object
## 1:坚定追求 pass-by-reference 的致命错误
在学习了 term20 后，程序员意识到pass-by-value 的开销，想要在一切场景下消除 pass-by-value，很容易传递一些 reference， 指向一些并不存在的对象。
本章主要讨论的是返回值是引用的情况。

## 2：当返回引用可能带来的陷阱
参考如下 代码
```C++
class Rational {
private:
    int numerator;   // 分子
    int denominator; // 分母

public:
    // 重载乘法运算符（返回临时值）
    Rational operator*(const Rational& other) const {
        return Rational(
            numerator * other.numerator,
            denominator * other.denominator
        );
    }
```
考虑上述代码，传递的是一个临时值，会调用这个对象的构造函数，构造该临时对象，又会通过赋值，调用被赋值对象的移动构造拷贝构造函数

<span style="font-weight: bold; font-style: italic; color: red;">
Notes：
</span>

```plaintext
移动构造函数的性能优化 主要体现在对资源（如动态内存、文件句柄等）的转移，而不是普通成员变量的赋值。具体来说：

1：指针/资源的高效转移（核心优化点）

直接“窃取”源对象的资源（如 data = other.data），避免深拷贝（如 new + memcpy）。

仅需 O(1) 时间（复制指针），而深拷贝需要 O(n) 时间（复制所有数据）。

2：成员变量（如 int、size_t，结构体）的赋值

像 size = other.size 这样的操作 不会带来性能提升，因为它只是简单的值复制（和拷贝构造函数一样）。但它的开销可以忽略不计（通常只是一个寄存器操作）。
```
在这里返回一个左值与返回一个左值的引用，对于调用该函数的函数是等效的，因为 reference 仅仅是一个别名


<span style="font-weight: bold; font-style: italic; color: black;">
如果要使得返回值传递引用，那么则必然要去创建内存空间
</span>

考虑如下代码

```C++
    Rational w(1, 2);
    Rational x(3, 5);
    Rational y = x * y;
```
x * y 结果是一个值为 3 / 10 的Rational 对象，* 运算符的返回的是一个引用，引用指向一个内存空间，必须由程序员自己创建这样的一个空间。创建这样的一个对象，或者是在stack上或者是在heap上。

<span style="font-weight: bold; font-style: italic; color: black;">
考虑在栈上创建内存空间：
</span>

```C++
friend const Rational& operator* (const Rational& r1, const Rational& r2)
{
        Rational temp;
        temp.numerator = r1.numerator * r2.numerator;
        temp.denominator = r1.denominator * r2.denominator;
        return temp;   // 注意，糟糕的代码
}
```
这是糟糕的代码，引用指向一个 local 变量，该函数结束，该变量的生命周期结束，引用指向的栈空间就已经释放了，造成了悬垂引用。
任何使用该函数返回的引用的操作，都会产生无定义行为。

<span style="font-weight: bold; font-style: italic; color: black;">
考虑在堆上创建内存空间：
</span>

```C++
friend const Rational& operator* (const Rational& r1, const Rational& r2)
{
    Rational *temp = new Rational();
    temp->numerator = r1.numerator * r2.numerator;
    temp->denominator = r1.denominator * r2.denominator;
    return *temp;
}
```
上述代码中，使用 new 创建了一个内存空间，由于是在堆上创建的，该对象的内存空间不会随着该变量的生命周期结束而结束，只能通过 delete 来释放，与此同时又带来了新的问题，什么什么时候调用 delete，程序员很容易忘记回收这样内存，又或者很难在合适的时机回收内存

考虑如下代码
```C++
    Rational w, x, y, z;
    w = x * y * z;
```
这个时候，调用了两次 * 操作，程序员如果非要获取 * 得到的指针，进行delete释放
如下所示
```
const Rational& temp1 = x * y;       // 第一次乘法
const Rational* ptr = &temp1;        // 获取指针
delete ptr;                          // 手动释放

const Rational& temp2 = temp1 * z;   // 第二次乘法
w = temp2;                           // 赋值
delete &temp2;                       // 再次手动释放
```
上述形式当然是可以的，但是

1：极其容易出错（稍不注意就会漏掉 delete）。

2：违反直觉：运算符重载通常不应要求用户手动管理内存。
 
使用 shared_ptr 解决上述内存泄漏的问题  理论上是可以的，但是形式上不美观，又会引入新的问题，还是使用值传递简单，形式统一。

我们注意到上述两种方案，不能奏效的原因是引用指向的临时对象调用了析构函数，或者是何时调用析构函数，什么方法调用析构函数容易出错且违反直觉。那么我们也很自然地进入下一条思路，使用 static 关键字来修饰局部变量，使得局部变量不需要被释放，然而这样又产生了新的问题。
<span style="font-weight: bold; font-style: italic; color: black;">
使用static关键字：
</span>

考虑如下代码
```C++
const Rational& operator*(const Rational& lhs, 
						  const Rational& rhs)
{	// warning, 又一堆烂代码
    static Rational result; //静态局部变量
 	result=...; //将lhs乘以rhs，然后将结果保存在result中
    return *result; //虽然编译器不报错，但是逻辑上是错误的
}

```
理想很美好，每一次调用都将运算的结果保存在static 变量中，这样的一种设计，立刻造成的是多线程安全性上的疑虑，它还有更深层次的瑕疵。

考虑如下代码
```C++ 

const Rational& operator*(const Rational& lhs, 
						  const Rational& rhs)
{	// warning, 又一堆烂代码
    static Rational result; //静态局部变量
 	result=...; //将lhs乘以rhs，然后将结果保存在result中
    return *result; //虽然编译器不报错，但是逻辑上是错误的
}

bool operator==(const Rational& lhs, 
				const Rational& rhs); // for Rationals
Rational a, b, c, d;
...
if ((a * b) == (c * d)) {
	//乘积相等，做相应动作
} else {
	//不相等，做相应动作
}

```
会发现无论是什么运算，== 运算的结果永远是true，虽然两次 * 运算将各自的运算结果保存到了局部变量 result 中，由于返回的他们返回的都是引用，因此调用端看到的永远是static Rational 对象的现值。

如果一个static不够，或许一个 static array 可行呢？
(1): 这又是一个错误的想法，首先必须选择array大小，如果n太小，可能会耗尽用以存储函数返回值的空间，又会陷入单一 static 设计中。
(2): 如果 n 太大，在该函数第一次调用的时候便调用 new [] ,调用了 n 次构造函数，函数所在进程结束的时候，调用n次析构函数。即便我们只调用一次 * 运算符，只需要一次result的存储。现在再去考虑运算结果的保存考虑，需要使用解引用后赋值，又会调用被赋值对象的复制构造函数，以及调用赋值对象的析构函数,但是问题一开始讨论的目的就是避免构造函数与析构函数，这样得不偿失。似乎可以去使用 std::vector  来动态扩容，避免大量的构造函数，析构函数的使用，但是对索引的管理，可读性与维护性又带来新的负担，这里暂时不去分析

## 3：总结
经过上述一系列的讨论，我们知道函数返回一个引用，总会带来各种各样的问题，我们的本意是减少构造函数与析构函数的调用，但是这样的一种设计，会带来各种各样内存泄漏，逻辑错误，逻辑复杂的潜在问题，也并没有减少构造函数或析构函数的使用。所以还是使用一种简单的方式，直接返回一个值，需要承受返回值的构造与析构成本，但是这是为了获得正确行为而付出的一个小小代价。
更重要的是，编译器可以对实现优化，因此在某些情况下，返回值的构造及析构函数会被消除，执行起来会比预期得快。
 












