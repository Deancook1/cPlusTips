# keyword：explict
explict 是C++的一个关键字，用于修饰构造函数或者是类型转换运算符，作用是禁止隐式转换，只允许显式调用。它的作用是防止意外的类型转换，提高代码的安全性

## 1: explict 修饰构造函数
### (1):默认隐式转换的问题
如果没有explict ，C++允许单参数构造函数进行隐式转换，可能导致意外的行为
示例：如果没有explict 的隐式转换
```C++
class MyString {
public:
    MyString(const char* str) {  // 允许隐式转换
        std::cout << "Constructor called: " << str << std::endl;
    }
};

void printString(MyString s) {
    // ...```
}

int main() {
    printString("Hello");  // 隐式转换：const char* → MyString
    return 0;
}
```
输出：
```
Constructor called: Hello
```
这里进行了隐式的类型转换，调用了 MyString 的构造函数。
当使用explict 修饰单参数构造函数的时候，

### (2):使用 explicit 禁止隐式转换
如果使用explict 修饰单参数构造函数，就必须显式构造对象，不能隐式转换
修正后的代码
```C++
class MyString {
public:
    explicit MyString(const char* str) {  // 禁止隐式转换
        std::cout << "Constructor called: " << str << std::endl;
    }
};

void printString(MyString s) {
    // ...
}

int main() {
    // printString("Hello");  // ❌ 错误！不能隐式转换
    printString(MyString("Hello"));  // ✅ 必须显式构造
    return 0;
}
```
输出：
```
Constructor called: Hello
```
现在必须显式调用 MyString("Hello")，避免了意外的隐式转换。

## 2：explict 修饰类型转换运算符
C++11 允许 explicit 修饰自定义类型转换，防止隐式转换。
示例：explicit operator bool()
```C++
class File {
public:
    explicit operator bool() const { return is_open(); } // 禁止隐式转换为 bool
    bool is_open() const { /* ... */ }
};

int main() {
    File f;
    // if (f) { ... }          // 错误：不能隐式转换为 bool（C++11 前允许）
    if (static_cast<bool>(f)) { // 正确：必须显式转换
        // ...
    }
}
```
这样可以避免 if (f) 这样的隐式转换，提高代码清晰度。

<span style="font-weight: bold; font-style: italic; color: red;">
Notes：要注意类型转换运算符与调用运算符的区别
</span>

```
类型转换运算符（Conversion Operator）:
operator TargetType() const;  // const 可选，取决于是否需要修改对象

函数调用运算符（operator()）:
ReturnType operator()(Args...);  // 可以接受任意参数
```