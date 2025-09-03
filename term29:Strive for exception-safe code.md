# Term29:Strive for exception-safe code
## 一：异常安全性的含义
程序的设计应该满足异常安全，而异常安全要求我们的程序在发生异常的时候，也处于一个合法的状态。异常安全有两个条件
1：不泄露任何资源
2：不允许数据破坏

```C++
class PrettyMenu {
public:
    ...
    void changeBackground(std::istream& imgSrc); // 改变背景图像
    ... 
private:
    Mutex mutex; 	// 这个对象的互斥量 
    Image* bgImage; 	// 当前背景图像
    int imageChanges; // 图像被改变的次数
};

void PrettyMenu::changeBackground(std::istream& imgSrc)
{
    lock(&mutex); 	// 取得互斥量(见条款14)
    delete bgImage; 	// 去除旧的背景
    ++imageChanges; 	// 更新图像变化计数
    bgImage = new Image(imgSrc); // 安装新的背景
    unlock(&mutex); 	// 释放互斥量
}

```
从异常安全性的角度来说，上述代码很糟糕
1：容易泄露资源，假如 bgImage = new Image(imgSrc); 发生了异常，被捕捉，那么永远不会执行unlock，那么互斥器就永远被把持住了
2：容易对数据进行破坏，假如 bgImage = new Image(imgSrc); 抛出异常，被捕捉，那么 imageChanges 就会永远的加1，但是其实新的图像没有被成功安装起来

## 二：资源泄露的解决方法
解决资源泄露很容易，参考条款13与条款14，使用RAII手法来确保互斥器及时释放。

```C++
void PrettyMenu::changeBackground(std::istream& imgSrc)
{
    Lock ml(&mutex); // 取得互斥量，并确保释放（见条款14）
    delete bgImage;
    ++imageChanges;
    bgImage = new Image(imgSrc);
}
```


## 三：解决数据的败坏

### 3.1 异常安全的三个层级
异常安全函数(Exception-safe funciton)提供以下三个保证之一，这三个保证都是针对异常前后数据状态前后不一致而言的：

1. 基本承诺:如果异常被抛出,程序内的任何事物仍保持在有效状态下.(一般指状态改变前后一致.例如不存在类似不男不女的可能,但无法对究竟是何种可能做任何假设)


2. 强烈保证:如果异常被抛出,程序状态不改变.(函数调用成功,就是完全成功.调用失败,程序保证回复到调用函数之前状态.类似于转账成功,完全成功。转账失败,回到转账之前的状态。)


3. 保证不抛出异常(nothrow):承诺绝不抛出异常。(这个可以通过关键字noexcept实现)目前可知的是内置类型都有nothrow的保证。nothropw的保证并不是说函数永远不会抛出异常，而是如果抛出异常，将是严重错误，会有意想不到的函数被调用。到底会不会抛出异常，是函数实现来决定的，noexcept 的保证仅是程序员的一种承诺，给编译器看的，让编译器有优化的空间，但是一旦发生异常，一定是很严重的错误，此时还不如把程序中断，要么遵守承诺，要么死亡。


异常安全码必须提供上述三种保障之一，我们的抉择是该如我们所写的代码提供哪一种保证？一般而言，我们会想着施加最强保证，但是我们很难在C part of C++中没有调用任何一个可能抛出异常的函数。例如任何使用动态内存的东西，例如所有STL容器，如果无法找到足够的内存以满足要求，便会抛出一个bad_alloc 异常，这是很难以避免的，所以大部分的抉择落在基本保证和强烈保证之间。


对于changeBackground 函数而言，提供强烈保证并不困难。
1. 首先，我们将PrettyMenu的bgImage数据成员的类型从内建的Image*指针替换为一种资源管理智能指针(见条款13)。说真的，对于防止资源泄露来说这绝对是一个好方法
2. 第二，我们对changeBackground中的语句进行重新排序，达到只有image被修改的时候才会增加imageChnages的目的。作为通用准则，一个对象的状态没有被修改就表明一些事情没有发生。

### 3.2 强烈保证的努力，针对上述代码的改进

#### 3.2.1 <span style="color: green; font-weight: bold;">改变代码逻辑，以及使用智能指针管理资源</span>
```C++
class PrettyMenu {
...
private:
    std::tr1::shared_ptr<Image> bgImage;
};
//修改后的PrettyMenu的成员函数
void PrettyMenu::changeBackground(std::istream& imgSrc) {
	Lock ml(&mutex);
	bgImage.reset(new Image(imgSrc));//以"new Image"的执行结果
	++imageChanges;//设定bgImage内部指针
}

```

注意这里已经足够好了，不需要再手动 delete 旧图像，因为这个图像已经被智能指针内部处理掉了，只有当这个指针被创建的时候才会调用reset 函数，如果从没进入那个函数也就不会使用delete。以对象管理资源，再次缩减了changeBackground  函数的长度
这里美中不足的是imgSrc,如果Image构造函数抛出异常，有可能输入流的读取记号已经被移走，而这样的搬移对程序其余部分是一种可见的状态改变，所以这个函数只提供基本的异常安全保障。

#### 3.2.2 <span style="color: green; font-weight: bold;">重要的策略：copy and swap</span>

有一个一般化的策略很经典会导致强烈保证，这个策略被称为copy and swap。当我们要改变一个对象时，先把它复制一份，然后去修改它的副本，然后在那个副本上做一切必要的修改，若有任何修改动作抛出异常，原对象仍保持未改变状态，改好了再与原对象交换。代码如下

```C++
struct PMImpl{
	std::shared_ptr<Image> bgImage;
	int imageChanges;
}

class PrettyMenux{
public:
	...
	void changeBackground(std::istream& imgSrc);
	...
private:
	Mutex mutex;
	std::shared_ptr<PMImpl> pImpl;
}
void prettyMenu::changeBackground(std::istream& imgSrc){
	// lock
	using std::swap;
	Lock m1(&mutex);
	// copy
	std::shared_ptr<PMImpl> pNew (new PMImpl(*pImpl));
	pNew->baImage.reset(new Image (imgSrc) );
	++pNew->imageChanges;
	// swap
	swap(pImpl,pNew);
}
```
这里实现上通常是将所有实际上通常是将所有“隶属对象的数据”从原对象放进另一个对象内，然后赋予原对象一个指针，指向那个所谓的实现对象（即副本）。这种手法被称为“pimpl idiom”，在条款31中会进行描述。




<div style="border-left: 4px solid #28a745; background-color: #f8fff8; padding: 12px; margin: 10px 0; border-radius: 4px;">
✅ <strong>简单说明什么是Pimpl Idiom</strong><br>参
Pimpl Idiom 是一种将类的实现细节（数据成员和私有函数）从一个头文件中分离出来，并将其放置在一个单独的实现类中的技术。在公有接口类中，仅保留一个指向这个实现类的（智能）指针。在这里的表现就是使用 pImpl 指向这个图像显示所有相关的资源。
</div>

#### 3.3.2  <span style="color: green; font-weight: bold;">单独论述copy and swap</span>
copy and swap 策略是对对象状态做出全有或全无改变的一个很好办法，但是一般而言它并不保证整个函数带有强烈的异常安全性，它有局限性，看如下的代码
```C++
void copySwapDoSomething(){
  ...//生成拷贝
  f1();
  f2();
  ...//拷贝回去
}

```
函数的异常安全性遵循木桶原理，即函数的异常安全性取决于它所调用操作的最弱异常安全性。假如f1不能提供强保证，那这个函数最多则只能提供基本保证。如果硬要保证它的强安全性，我们需要在调用f1之前把程序的状态存档，抓住f1抛出的所有异常，然后恢复程序原来的运行状态，显然非常麻烦不太实际。
如果f1 f2 都是强烈异常安全，情况并不好转，因为毕竟如果 f1 圆满结束，程序状态在任何方面都可能有所改变，因此，如果 f2 随后抛出异常，程序状态和 somefunction 被调用之前并不相同，甚至当 f2 没有改变任何东西时候也是如此。

问题出现在“连带影响”，如果由函数只操作局部状态，便相对容易的提供强烈保证，但是函数对“非局部性数据”有连带影响时，提供强烈保证就困难的多。例如，如果调用 f1 带来的影响是某个数据库被改动了，那就很难让someFunc具备强烈安全性。另一个主题是效率。copy-and-swap得好用你可能无法（或不愿意）供应的时间和空间。所以，“强烈保证”并不是在任何时候都显得实际。
另一个主题是效率，copy-and-swap 的关键在于修改对象数据的副本，然后在一个不抛出异常的函数中将修改后的数据和原件置换，因此必须为每一个即将被改动的对象做出一个副本，那得耗用你可能无法或无意愿供应的时间和空间。

<span style="color: red; font-weight: bold;">当强烈保证不切实际时，你就必须提供基本保证。对于很多函数，提供异常安全性之基本保证是一个绝对通情达理的选择。</span>

如果你写的函数完全不提供异常安全保证，情况又有点不同，因为他人可以合理假设你在这方面有缺失（认为你能力不足）。
一个软件系统要不就具备异常安全性，要不就全然否定。只要有一个调用的函数不具备异常安全性，那么整个系统都不具备异常安全性。很多老旧代码并不具备异常安全性，所以今天很多系统都不能称为具有异常安全性的。

## 四：总结
1. 尽量写出满足异常安全的代码，（1）应该避免资源泄漏，（2）在异常发生的时候，应该避免数据败坏
2. 避免资源泄露，可以通过使用对象来管理资源，自动管理对象的生命周期
3. 避免数据败坏，可以通过改善代码逻辑，copy-and-swap 方法来实现，copy-and-swap 方法应该考虑实现复杂度与效率。
4. 即便如此，想要去实现异常安全代码也是一个很复杂的操作，因此，对于很多函数，提供异常安全性之基本保证是一个绝对通情达理的选择。