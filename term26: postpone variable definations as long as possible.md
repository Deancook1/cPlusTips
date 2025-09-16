# term26: Postpone variable definitions as long as possible

## 1:Background
每当定义一个变量时，就会带来构造和析构的运行成本，因为代码运行到定义时会调用对象的构造函数，当离开作用域时便会调用析构函数。那么我们可能会直接想到，不定义未使用的变量不就可以了吗? 但这只是一部分，有时候定义了要使用的变量依然有几率会导致浪费。

## 2: 变量定义的时机
参考如下代码
```C++
std::string encryptPassword(const std::string& password){
  using namespace std;
  string encrypted; //定义太早了
  encrypted = password;

  if(password.length() < minPasswordLength)
    throw logic_error("Password too short");
  ....//加密
  return encrypted;
}
```
字符串对象encrypted显然不是我们所讲的"未使用的变量"，因为下面的代码明显要用到它。但如果这个函数抛出了异常，那它就是一个未使用变量了，它所消耗的资源也都浪费了。因此我们需要把它的定义尽量往后推迟，直到我们100%确定要用到:

```C++
std::string encryptPassword(const std::string& password){
  using namespace std;

  if(password.length() < minPasswordLength)
    throw logic_error("Password too short");

  string encrypted;  //这时我们才确定它会被用到
  encrypted = password;
  ....//加密
  return encrypted;
}
```

仍然存在优化的空间，注意上面encrypted是默认构造的，因为没有任何参数传进构造函数。条款4 讲到默认构造一个对象再进行赋值是比较低效的，我们应该直接使用带有参数的构造函数:

```C++
//用:
string encrypted(password);
//替换掉:
string encrypted;
encrypted = password;
```

<div style="border-left: 4px solid #28a745; background-color: #f8fff8; padding: 12px; margin: 10px 0; border-radius: 4px;">
✅ <strong>可以看到推迟定义的意义</strong><br>
不仅仅需要把变量的定义推迟到100%要用到的地方，还要把它推迟到100%有构造参数可用的时候。这样做既可以避免不必要的构造和析构过程，也能节省默认构造再赋值的成本。而且这样的代码也更可读，因为变量定义在了真正需要它的环境下。
</div>


## 3:循环场景
```C++
//代码A，在外面定义
Widget w;
for(int i=0; i<n, i++){
  w=...;
  ...
}

//代码B，在里面定义
for(int i=0; i<n; i++){
  Widget w(...);
  ...
}
```
那么我们就来分析一下A和B各自的运行成本:
```
A: 1个构造 + n个赋值 + 1个析构
B: n个构造 + n个析构
```

<div style="border-left: 4px solid #28a745; background-color: #f8fff8; padding: 12px; margin: 10px 0; border-radius: 4px;">
✅ <strong>可以看到推迟定义的意义</strong><br>
现在我们就可以看出来了，对于赋值成本低于(构造+析构)的类，A是更高效的选择，尤其是当n很大的时候。反之如果赋值成本大于(构造+析构)，B则是更好的选择。但是对象在A的作用域比在B要大，有时是不利于程序的可读性和可维护性的。因此除非你知道赋值成本低于(构造+析构)，而且这段代码要更注重效率，那么我们应该默认使用B。
</div>


## 4：总结
这个章节体现出一种逐步优化的过程。
1. 首先避免变量过早定义，避免进入不需要变量的分支，等到一定要用到该变量的时候再去定义。
2. 避免变量进入不必要的默认构造函数，等到有了可以供变量进行初始化的实参再去定义，同时可读性更高。
3. 循环的场景，则需要根据 赋值操作 与（构造加析构）的成本对比，来确定是在循环内部进行变量的定义还是在循环外进行变量的定义。