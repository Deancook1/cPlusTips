
# Term22:Declare data member
## 一：为什么不将成员变量声明为 public
### 1.1 语法一致性，思维比较自然
访问变量只能通过函数，不需要再纠结访问变量的写法
### 1.2 可以通过函数对成员变量进行更细致的权限管理
```C++
class AccessLevels {
public:
    ...
    int getReadOnly() const { return readOnly; }
    void setReadWrite(int value) { readWrite = value; }
    int getReadWrite() const { return readWrite; }
    void setWriteOnly(int value) { writeOnly = value; }
private:
    int noAccess; 	// 不能访问这个int
    int readOnly; 	// 对这个int的只读访问
    int readWrite; 	// 对这个int的读写访问
    int writeOnly; 	// 对这个int的只读写访问
};

```
### 1.3 C++中将成员变量隐藏在函数接口的背后，可以为所有可能的实现提供弹性
将成员变量隐藏在函数接口后边，可以为“所有的实现”提供弹性。可能函数实现的方法已经改变，而用户最多只需要重新编译


封装比它起初看起来要重要，因为只能去使用对象提供的函数去影响变量，函数由对象提供，一定满足了这个对象对这个变量的约束。进一步来说，你保留了日后对实现决策进行变动的权利，如果使用public 声明变量，那么对这个变量的改动是极度受限的，大量使用这个变量的逻辑可能需要修改，影响客户端，很多操作不能执行，

## 二：最好也不要把成员变量声明为protected

protected数据成员不是有比public数据成员更好的封装性么？令人感到吃惊的回答是，它们不是。
标记了protected 的变量，意味着可以被子类使用，那么如果这个变量发生了修改，对子类的逻辑也会造成很大的影响。

<div style="border-left: 4px solid #28a745; background-color: #f8fff8; padding: 12px; margin: 10px 0; border-radius: 4px;">
✅ <strong>不同继承方式的意义</strong><br>
继承方式是对基类成员的一次"降权处理"，目的是让派生类能够根据实际情况，重新定义哪些功能应该对外暴露，哪些应该继续隐藏。

</div>

## 三：牢记
1.  切记将成员变量声明为private。这可赋予客户访问数据的一致性、可细微性划分访问控制、允许约束条件获得保证，并提供class作者以充分的实现弹性

2.  protected并不比public更具封装性