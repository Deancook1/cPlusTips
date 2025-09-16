# **term20:Prefer pass-by-reference to pass-to-value**

## 1：使用 pass-by-value 的开销分析
缺省情况下，即没有显式指定传递方式的时，C++以by value的方式 传递对象至函数。
函数传参以及函数返回返回值，都是使用得上述方式。
    1：当使用值传递的时候，函数参数得到的是实参的一个副本
    2：调用者调用函数得到的也是返回值的一个副本。
上述两种情况都会调用对象的拷贝构造函数。

考虑以下代码
```C++
class Person {
public:
    Person();
	virtual ~Person();
private:
    std::string name;
    std::string address;
};
 
class Student :public Person {
public:
    Student();
    ~Student();
private:
    std::string schoolName;
    std::string schoolAddress;
};

bool validateStudent(Student s);

Student plato;          //定义Student对象
bool platoIsOK  = validateStudent(plato); //传值调用

```
毫无疑问在调用 validateStudent 函数的时候，传参使用的是值传递，函数调用结束，形参会被销毁。
现在我们考虑一下一次调用该函数的开销：Person中有两个string 对象，Student 有两个string，构造的时候首先调用student 的拷贝构造函数，再去调用两个string的构造函数，再去调用Person 的拷贝构造函数，再去调用Person中两个对象的拷贝构造函数。两次对象的构造函数调用，四次string的构造函数调用。
当形参S要去释放的时候，先去调用Student的析构构造函数，两个string的析构函数，再去调用Person的析构函数，再去调用Person中两个string的析构函数。这样算下来，共调用了六次构造，六次析构。

## 2：使用 pass-by-reference 的好处
### 2.1：避开了多次拷贝构造函数
开销是很大的，如果我们想避开上述过程，那就要去使用pass by reference-to-const，这种方式效率高得多，因为在传参的时候没有任何对象需要被构造。const 是重要的，这里使得调用者知道传递引用进去的参数，不会被改变。

### 2.2：可以避免 slicing（对象切割问题）
以by reference 的方式传递参数，还可以避免slicing 问题，当一个derived class 对象以by value 方式传递并被视为一个base class 对象，base class 的copy构造函数会被调用，造成此对象的行为像个derived class 的那些特化性质都被切割掉了，仅仅留下一个base class 对象。

解决切割问题，最好的办法就是使用by referrence-to-const 方式传递参数。

## 3：小型types 都应该使用 pass-by-value 驳斥
如果窥探编译器的底层，我们可以知道传递引用的底层其实是传递了一个指针。而对于一些内置类型，和STL中的迭代器和函数对象，由于他们比较小，所以使用pass-by-value不无道理。

但是所有的小型types 都应该使用pass-by-value，是个不可靠的推论，因为
1：对象小不意味着其copy 构造函数不昂贵，许多对象，包括大多数STL容器，内含的东西并不多，只比一个指针多些，但复制这种对象却需要承担复制那些指针所指向的每一样东西，那将非常昂贵，可看下列代码
```C++
deque(const deque& __x) : _Base(__x.get_allocator(), __x.size()) 
    { uninitialized_copy(__x.begin(), __x.end(), _M_start); }
``` 
上述是一个deque的构造函数，参照STL源码，假如没有使用 const 引用的方式那么将某一个deque对象传参到一个deque的构造函数的时候，形参的生成需要调用拷贝构造函数，这里的拷贝构造函数将将deque中的迭代器指向的内存依次复制，开销是很大的

2：即便内置类型和自定义类型有相同的底层表述，编译器倾向于把内置类型存放在寄存器中，而我们知道by reference 其实是传递了一个指针，编译器会将这个指针传进寄存器中，也提高了运行效率。

3：一个用户自定义类型，其大小容易有所改变，一个 type 目前虽然小，但是将来可能会会变大，就比如引用了不同版本的库，string类型可能大七倍。

一般而言，pass-by-value只适合于内置类型，STL迭代器类型，和小型函数对象。内置类型一般字节数很少，拷贝开销很小，STL迭代器类型本质上是轻量级指针，拷贝成本低，函数对象通常比较小巧，拷贝开销都不大。

## 4：使用 pass-by-reference 的缺点 
但是并不是所有场景都适合使用pass-by-reference，有其缺点
1：传递引用，其实传递是指针，在函数内部使用的时候，需要解引用，多一次内存寻址
```C++
void foo(int& x) { x += 1; }  // 需要解引用
void bar(int x)  { x += 1; }  // 直接操作寄存器
```
对 int、double 等内置类型，按值传递可能比按引用 更快（尤其在频繁调用的场景）。

2:值传递更加直观，且得到的是实参的副本，无需担心对象生命周期问题。




