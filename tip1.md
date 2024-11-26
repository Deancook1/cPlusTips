# question:
为什么在已经有了函数指针，匿名对象的情况下，仍然需要std::function?  

# answer:
	1. std::funciton 本质上是一个对象，可以认为是封装了一个指针加一些上下文数据。
    2. 函数指针仅仅是一个指针，指向了一个代码片段，因此他只能去操作全局变量或者是局部变量
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
            
       
    

	
