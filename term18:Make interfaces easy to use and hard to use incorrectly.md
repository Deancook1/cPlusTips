# Term18 : Make interfaces easy to use and hard to use incorrectly
## 1:background
接口是用户与代码沟通的手段，假如用户调用接口，用法不正确，编译报错，或者是产生了意料之外的错误。至少也说明，这个接口设计的不好。
## 2：错误类型
### 2.1 容易产生的错误
```C++
class Date
{
public:
    Date(int month, int day, int year);
    // ...
};
```
假设有一个上述用来表现日期的class构造函数，我们可能会使用如下的构造方式
```C++
Date d(30, 3, 1995);    // 应该是3, 30而不是30,3，很容易输错顺序，导致意义错误
```
```C++
Date d(2, 30, 1995);   // 应该是3, 30而不是2, 30，2和3紧挨着，很容易误触碰，导致隐藏的bug
```
### 2.2 很多客户端错误可以因为引入新类型而得到预防

```C++
struct Day
{
    explicit Day(int d) : val(d)
    {
    }
    
    int val;
};

struct Month
{
    explicit Month(int d) : val(d)
    {
    }
    
    int val;
};

struct Year
{
    explicit Year(int d) : val(d)
    {
    }
    
    int val;
};

class Date
{
public:
    Date(const Month &m, const Day &d, const Year &y);
    // ...
};

Date d(30, 3, 1995);    // 错误！不正确的类型
Date d(Day(30), Month(3), Year(1995));    // 错误，不正确的类型
Date d(Month(3), Day(30), Year(1995));    // OK，类型正确

```
令Day、Month、Year成为成熟且经充分使用的class并封装其内数据，比简单使用上述struct好（见条款22）。但即使上例也已经足够示范：明智而审慎地导入新类型，以新的类型来作为函数的参数，对预防“接口被误用”有神奇疗效。

### 2.3 引入新的类型后，对实际传参的或者是返回值类型进行限制是合理的
#### 2.3.1 使用函数来限制函数的值
例如，限制上述类型的值是合理的
例如一年只有12个有效月份，所以Month应该反映这一事实。办法之一是利用enum表现月份，但enum不具备我们希望拥有的类型安全性，例如enum可被拿来当一个int使用（见条款2）。比较安全的解法是预先定义所有有效的Month：
```C++
class Month
{
public:
    static Month Jan()    // 函数，返回有效月份，稍后解释为什么使用函数而非对象
    {
        return Month(1);
    }
    
    static Month Feb()
    {
        return Month(2);
    }
    // ...
    static Montn Dec()
    {
        return Month(12);
    }
    // ...    其他成员函数

private:
    explicit Month(int m);    // 阻止生成其他月份，这是private的
    // ...
};

```
#### 2.3.2 限制类型内什么事可做，什么事不能做
常见的限制是加上const
例如条款3曾经说明为什么“以const修饰operator *的返回类型”可阻止客户因“用户自定义类型”而犯错：
```C++
if (a * b = c)   // 原意是要做一次比较动作，少写了一个=
```
如上自定义 * 运算符的结果只能是const类型，那就不能有元素对其结果赋值，避免了可能的失误

### 2.4 除非有好理由，否则应该尽量令你的types的行为与内置types一致
客户已经知道像int这样的type有什么行为，所以你应该努力让你的type在合样合理的前提下也有相同表现。例如，如果a和b都是int，那么对a * b赋值并不合法，所以除非你有好的理由与此行为分道扬镳，否则应该让你的types也有相同的表现。使得，一旦怀疑，就请拿int做范本。

避免无端与内置类型不兼容的真正理由是为了提供行为一致的接口。很少有其他性质比得上“一致性”更能导致“接口容易被正确使用”，也很少有其他性质比得上“不一致性”更加剧接口的恶化。STL容器的接口十分一致（虽然不是完美地一致），这使它们非常容易被使用。例如每个STL对象都有一个名为size的成员函数，它会告诉调用者目前容器内有多少对象。与此对比的是Java，它允许你针对数组使用length property，对String使用length method，而对List使用size method；.NET也一样混乱，其Array有个property名为Length，其ArrayList有个property名为Count。有些开发人员会以为整合开发环境（integrated development environment，IDE）能使这般不一致性变得不重要，但他们错误。不一致性对开发人员造成的心理和精神上的摩擦与争执，没有任何一个IDE可以完全抹除。

<span style="font-weight: bold; font-style: italic; color: red;">
这一条最重要的目的就是降低程序员的心智负担
</span>

### 2.5 任何接口如果要求客户必须记得做某些事情，就是有着“不正确使用”的倾向，因为客户可能会忘记做那件事

例如
```C++
Investment *createInvestment();    // 来自条款13，为求简化暂略参数
```
为避免资源泄露，createInvestment返回的指针最终必须被删除，但那至少开启了两个客户错误机会：没有删除指针，或删除同一个指针超过一次。

条款13表明客户如何将createInvestment的返回值存储于一个智能指针如auto_ptr或tr1::shared_ptr内，因而将delete责任推给智能指针。但万一客户忘记使用智能指针怎么办？许多时候，较佳接口的设计原则是先发制人，就令factory函数返回一个智能指针：

```C++
std::tr1::shared_ptr<Investment> createInvestment();
```
这便实质上强迫客户将返回值存储于一个tr1::shared_ptr内，几乎消弭了忘记删除底部Investment对象（当它不再被使用时）的可能性。

### 2.6 假设设计了某个接口，让其有某种预期行为，对其传参的某个class进行操作，还不如在该class初始化的时候，直接绑定。
假设class设计者期许那些“从createInvestment取得Investment *指针”的客户将该指针传递给一个名为getRidOfInvestment的函数，而不是直接在它身上动刀（即使用getRidOfInvestment函数回收资源，而不是直接delete，“直接在它身上动刀”指使用delete）。这样一个接口又开启通往另一个客户错误的大门，该错误是“企图使用错误的资源析构机制”（即拿delete替换getRidOfInvestment）。createInvestment的设计者可以针对此问题先发制人：返回一个“将getRidOfInvestment绑定为删除器（deleter）”的tr1::shared_ptr。

tr1::shared_ptr提供的某个构造函数接受两个实参：一个是被管理的指针，另一个是引用次数变成0时将被调用的“删除器”。这启发我们建立一个null tr1::shared_ptr并以getRidOfInvestment作为其删除器，像这样：
```C++
std::tr1::shared_ptr<Investment> pInv(0, getRidOfInvestment);    // 企图创建一个null shared_ptr
                                                                 // 并携带一个自定的删除器，此式无法通过编译
```


但上例不是有效的C++。tr1::shared_ptr构造函数坚持其第一个参数必须是个指针，而0不是指针，是个int。是的，它可被转换为指针，但在此情况下并不够好，因为tr1::shared_ptr坚持要一个不折不扣的指针。转型（cast）可以解决这个问题：
```C++
// 建立一个null shared_ptr并以getRidOfInvestment为删除器
// 条款27提到static_cast
std::tr1::shared_ptr<Investment> pInv(static_cast<Investment *>(0), getRidOfInvestment);

```
因此，如果我们要实现createInvestment使它返回一个tr1::shared_ptr并夹带getRidOfInvestment函数作为删除器，代码看起来像这样
```C++
std::tr1::shared_ptr<Investment> createInvestment()
{
    std::tr1::shared_ptr<Investment> retVal(static_cast<Investment *>(0), getRidOfInvestment);
    retVal = ...;    // 令retVal指向正确对象
    return retVal;
}
```
当然啦，如果被pInv管理的原始指针（raw pointer）可以在建立pInv之前先确定下来，那么“将原始指针传给pInv构造函数”会比“先将pInv初始化为null再对它做一次赋值操作”更好




tr1::shared_ptr有一个特别好的性质是：它会自动使用它的“每个指针专属的删除器”，因而消除另一个潜在的客户错误：所谓的“cross-DLL problem”。这个问题发生于“对象在动态链接程序库（DLL）中被new创建，却在另一个DLL内被delete销毁”。在许多平台上，这一类“跨DLL的new/delete成对运用”会导致运运行期错误。

<span style="font-family: 'Microsoft YaHei', sans-serif; font-weight: bold; color: red;">
CROSS-DLL problem 详解
</span>

```
跨DLL的new/delete问题（Cross-DLL Problem）发生在：

对象在DLL A中使用 new 创建（分配内存）。

却在DLL B中使用 delete 销毁（释放内存）。

由于不同DLL可能使用不同的运行时库（CRT, C Runtime Library），导致内存分配和释放的堆管理器不一致，最终引发运行时错误（如崩溃、内存泄漏）。
```
r1::shared_ptr没有这个问题，因为它缺省的删除器是来自“tr1::shared_ptr诞生所在的那个DLL”的delete。这意思是，举例来说，如果Stock派生自Investment而createInvestment实现如下：

```
std::tr1::shared_ptr<Investment> createInvestment()
{
    return std::tr1::shared_ptr<Investment>(new Stock);
}

```
返回的那个tr1::shared_ptr可被传递给任何其他DLL，无需在意“cross-DLL problem”。这个指向Stock的tr1::shared_ptr会追踪“当Stock的引用次数变成0时该调用的那个DLL’s delete”。

本条款并非特别针对tr1::shared_ptr，而是为了“让接口容易被正确使用，不容易被误用”而设。但由于tr1::shared_ptr如此容易消除某些客户错误，值得我们核计其使用成本。最常见的tr1::shared_ptr实现品来自Boost（见条款55）。Boost的shared_ptr是原始指针的两倍大，以动态分配内存作为簿记用途和“删除器之专属数据”，以virtual形式调用删除器（假如是一个指向派生类的基类指针，也能正确删除所指对象），并在多线程程序修改引用计数时蒙受线程同步化（thread synchronization）的额外开销（只要程序员定义一个预处理器符号就可以关闭多线程支持）。总之，它比原始指针大且慢，而且使用辅助动态内存。在许多应用程序中这些额外的执行成本并不显著，然而其“降低客户错误”的成效却是每个人都看得到。


## 3:总结
1.好的接口很容易被正确使用，不容易被误用。你应该在你的所有接口中努力达成这些性质。

2.“促进正确使用”的办法包括接口的一致性，以及与内置类型的行为兼容。

3.“阻止误用”的办法包括建立新类型、限制类型上的操作，束缚对象值，以及消除客户的资源管理责任。

4.tr1::shared_ptr支持定制型删除器（custom deleter）。这可防范DLL问题，可被用来自动解除互斥锁（mutex，见条款14）等等。