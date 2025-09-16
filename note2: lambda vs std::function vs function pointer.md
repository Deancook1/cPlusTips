# question:
为什么在已经有了函数指针，匿名对象的情况下，仍然需要std::function?  

# answer:
	1. std::funciton 本质上是一个对象，可以认为是封装了一个指针加一些上下文数据。
    2. 函数指针仅仅是一个指针，指向了一个代码片段，因此他只能去操作全局变量或者是局部变量,std::function 可以去调用一个可调用对象，这个对象可以是一个类的实例，对象中封装好了数据，也可以是一个匿名函数，匿名函数通过捕捉到了参数，保存了下来。std::function 可以去操作的数据多种多样。
    3. std::function  是一个模版类，与STL的思想相统一，
    4. 使用std：：function 可以实现多态，见如下代码片段

        #include <iostream>
        #include <functional>
        #include <vector>
        #include <memory>

        // 多态基类
        class Base {
        public:
            virtual ~Base() = default;
            virtual void execute() const = 0;  // 虚函数
        };

        // 派生类1
        class Derived1 : public Base {
        public:
            void execute() const override {
                std::cout << "Derived1::execute() called" << std::endl;
            }
        };

        // 派生类2
        class Derived2 : public Base {
        public:
            void execute() const override {
                std::cout << "Derived2::execute() called" << std::endl;
            }
        };

        int main() {
            // 使用 std::function 存储 Base 类的指针（或智能指针）
            std::vector<std::function<void()>> functions;

            // 创建派生类对象，并将其添加到 vector 中
            std::shared_ptr<Base> obj1 = std::make_shared<Derived1>();
            std::shared_ptr<Base> obj2 = std::make_shared<Derived2>();

            // 使用 lambda 表达式将 execute 方法绑定到 std::function
            functions.emplace_back([obj1]() { obj1->execute(); });
            functions.emplace_back([obj2]() { obj2->execute(); });

            // 调用存储的函数
            for (const auto& func : functions) {
                func();
            }

            return 0;
        }

    5. 使用std::function 相比于函数指针，必然会引入一些开销
    6. 匿名函数只是一种简便的方法，编译器会自动地将他编译为一个对象。匿名函数的定义域范围只是他的定义范围，若脱离这个定义范围，会自动地进行释放。
       匿名函数不能作为全局变量使用因为，匿名函数会有比如捕捉列表，捕捉列表会捕捉上下文语义中的数据。如果将匿名函数定义为全局变量，那么可能该捕捉范围内的数据已经释放，但是该匿名函数所代表全局变量仍然存在，这样对内存安全有风险，所以匿名函数只是一种简便的开销较小的方法，如果想使用更高级的功能，可以使用std::function
    7. 对于不同的可调用对象，std::function 提供了统一的接口。
            
       
    

	
