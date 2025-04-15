# API

### void* 
1. void* 是一个无状态指针（泛型指针），可以指向任意的数据类型，但是不包含数据的类型信息。
2. 不能直接使用*Ptr对无状态指针解引用，必须先显示转换-> *(static_cast<int*>)(ptr)
3. ptr++错误，因为不知道节点类型
- 用途：用来记录分配内存的首地址，malloc(10 * sizeof(int))返回一个void*指针

### operate new，operator delete 这两者需要配对使用实现手动内存回收
operator new 是一个底层的内存分配函数，用于动态分配原始内存。它不直接调用构造函数，而是单纯分配指定大小的内存块.！！但会在头部记录一个元数据（如分配大小、对齐信息）
- new 表达式 → 调用 operator new 分配内存 → 调用构造函数。
- void* operator new(size_t size); // 默认版本
- 手动分配内存实现：
``` c++
	void* newBlock = operator new(BlockSize_); // 分配了BlockSize_大小的内存块，newBlock是起始地址
	Slot* firstSlot  = reinterpret_cast<Slot*>(newBlock); // 把newBlock前几个字节转换为Slot类型的指针
	Slot* current = firstSlot;
	for (int i = 0; i < count - 1; ++i) {
		current->next = current + 1; // 下一个 Slot
		current++;
	}
````
operator delete 接受一个void*指针，但是void*是无状态指针因此不知道需要释放多大内存空间，这里会根据地址偏移后获得元数据，里面包含了分配内存大小等信息，下面是一个样例。
````c++
class MyClass {
public:
    void* operator new(size_t size) {
        void* ptr = malloc(size + sizeof(size_t));
        *static_cast<size_t*>(ptr) = size; // 记录大小到元数据
        return static_cast<char*>(ptr) + sizeof(size_t);
    }
    
    void operator delete(void* ptr) {
        void* base = static_cast<char*>(ptr) - sizeof(size_t);
        size_t size = *static_cast<size_t*>(base);
        free(base); // 根据记录的元数据释放内存
    }
};
````

### 析构函数
析构函数是在对象生命结束时被调用，用来清理对象的资源，然后内存会被自动回收。！但是显示调用析构函数并不会马上自动回收内存
- 手动内存回收
````c++
	if (p) { // ？？为什么调用析构后还要归还内存？ 
		// 显示调用析构函数会执行对象内部的清理逻辑（如释放成员资源、关闭文件等），但不会释放对象自身占用的内存。
		p->~T();
		// operator delete(p); // 手动内存回收
		HashBucket::freeMemory(reinterpret_cast<void*>(p), sizeof(T));//自定义回收
	}
````

### C++强制类型转换
- static_cast 在编译时确定转换类型（无运行时类型检查），用于基本数据类型、向上转换、void*->int*->void*
- dynamic_cast 主要用于多态类对象->完整类对象的向下转换或者void*，有运行时类型检查，若是指针返回nullptr，引用返回std::bad_cast; 
- reinterpret_cast 用于：1.指针<->整数（内存地址操作） 2. 不同指针/引用任意转换
````c++
class Base { virtual void foo() {} }; // 多态基类
class Derived : public Base {};

// static_cast: 向上转换和基本类型转换
Derived d;
Base* b1 = static_cast<Base*>(&d);      // 安全
int i = 10;
double d = static_cast<double>(i);      // 安全

// dynamic_cast: 向下转换（运行时检查 指针指向的类对象是否可以转换为目标类对象）
Base* b2 = new Derived;  // new Derived -> Base* -> Derived*
Derived* p = dynamic_cast<Derived*>(b2); // 成功
Base* b3 = new Base; // new Base -> Base*  不能-> Derived*
Derived* p2 = dynamic_cast<Derived*>(b3); // 返回nullptr ！！！！

// reinterpret_cast: 无关类型转换
int x = 42;
int* p = &x;
char* c = reinterpret_cast<char*>(p);  // 将int*转char*
uintptr_t addr = reinterpret_cast<uintptr_t>(p); // 指针转整数
````

### 进程修改知识
#### 内存模型和原子变量
- std::memory_order_relaxed 无同步约束
- std::memory_order_consume 比acquire屏障等级低，只保证依赖该原子变量的数据不会暴露在之前，并不好用
- std::memory_order_acquire 后续操作不会排序到该指令之前
- std::memory_order_release 前面的操作不会排序到该指令之后
- std::memory_order_acq_rel 可以看成是一个分水岭，完全隔离前面与后面的指令顺序
- std::memory_order_sql_cst 绝对顺序执行，虽然安全但是效率低

- 理解： 原子变量需要搭配内存模型使用，使用过的有下面方法
1. 使用std::atomic<xxx> a;结构体对象可通过a.load()读取，a.store()设置，并且通过compare_exchange_weak/strong 判别能否修改,weak允许伪失败（即预期值等于当前值也会失败）具有较高性能适合循环；而strong要求要么成功要么失败，性能低适合非循环条件
2. 使用std::atomic_flag a;结构体对象，该结构体内部有atomic<bool> _Storage变量可用来记录是否上锁，可通过test_and_set(std::memory_order_acquire)尝试上锁,并记得clear释放锁
- 代码：
````c++
class A {
public:
	A(int a) :a(a) {

	}
	std::atomic<int> a;
};

std::atomic<A*> a =new A(2);
A* old = a.load(std::memory_order_relaxed);
A* p = new A(123);
p->a.store(123 + old->a,std::memory_order_relaxed);
a.compare_exchange_weak(old, p, std::memory_order_release, std::memory_order_relaxed);
std::cout << a.load(std::memory_order_acquire)->a << std::endl;

std::atomic_flag flag;
if (flag.test_and_set(std::memory_order_acquire))std::cout << "Yeah!" << std::endl;
flag.clear(std::memory_order_release);
````

#### 互斥锁mutex与lock_guard
理解： 通过实例化mutex类创建锁对象，然后通过创建lock_guard类对象进行上锁操作，若上锁失败则会进入阻塞状态直至成功获得锁。互斥锁会锁住一个代码块内互斥操作。
- 例子：分别使用互斥锁和CAS操作实现
````c++
// 互斥锁
std::mutex mutex_;
auto func = [&](std::string s) {
	std::lock_guard<std::mutex> lock(mutex_);
	std::cout << s << std::endl;
	};
	
// CAS操作
std::atomic_flag flag_;
auto func1 = [&](std::string s) {
	while (flag_.test_and_set(std::memory_order_acquire)) {
		std::this_thread::yield();
	}
	try {
		std::cout << s << std::endl;
	}
	catch (...) {
		flag_.clear(std::memory_order_release);
		throw;
	}
	flag_.clear(std::memory_order_release);
	return;
	};

int n = 1000;
std::vector<std::thread> tlist;
for (int i = 0; i < n; i++) {
	tlist.emplace_back(func1, std::to_string(i));
}
for (auto& t : tlist)t.join();
````

#### std::Thread中join()与detach区别
- join用于`主子线程`之间的同步(因为调用必须知道目标线程对象)，调用线程会阻塞直到目标子线程执行完毕。

- detach用于主线程与子线程分离，子线程转为后台运行，主线程继续执行而不等待。
1. 需要确保子线程不访问主线程的局部变量或已释放资源（开启子线程的同作用域内变量以及回收过的资源）

#### 使用__thread(c/c++) 和 thread_local(c++风格) 来定义变量为线程私有，常用来确保变量在线程之间独立

#### lock_guard mutex unique_lock 
- lock_guard执行上锁操作，对象一旦创建就会对上锁
- mutex是互斥锁也能进行.lock .unlock 操作，但是需要手动释放锁，并且无异常处理机制，一旦异常就会一直阻塞 
- unique_lock 在lock_gaurd基础上多了.lock .unlock操作，而且有异常安全机制和自动释放锁资源

#### sem_t 信号量
定义：sem_t是 POSIX 线程库（<semaphore.h>）中定义的信号量类型，用于多线程或多进程间的同步操作。信号量是一个整型计数器，通过原子操作（sem_post 增加、sem_wait 减少）实现线程间的协调。

使用方法：
````c++
	sem_t sem; 
    sem_init(&sem, false, 0);                                               // false指的是 不设置进程间共享
    // 开启线程
    thread_ = std::shared_ptr<std::thread>(new std::thread([&]() {
        tid_ = CurrentThread::tid();                                        // 获取线程的tid值
        sem_post(&sem);														// 信号量值 +1，唤醒主线程
        func_();                                                            // 开启一个新线程 专门执行该线程函数
    }));
    sem_wait(&sem); // 主线程在次阻塞等待子线程sem_post(&sem);
````

### explicit
用于修饰类的构造函数,被修饰的构造函数的类，不能发生相应的隐式类型转换，只能以显示的方式进行类型转换。
````c++
class A{
	A(double a){};
	A(const A& a){};
}
// 下面是隐式构造
A a1 = 3;
A a2 = 3.2;
A a3 = a2; // 隐式调用拷贝构造

// 加上explicit 关键字后 explicit A(xxx xxx){};
A a1(3) 报错
A a1(3.2) 正确
A a2(a1) 正确

假如没有重载=运算符，那么一切使用=创建对象的过程都会报错
A a1 = 3;
A a2 = 3.2;
A a3 = a1;
````

### Lambda表达式
理解： 可以认为是匿名方法，其最大的特点就是变量捕获。变量捕获分为值捕获和引用捕获，值捕获发生在定义时，而引用捕获发生在调用时。

### std::function<>与函数指针
- 区别：单纯的函数指针指向函数的首地址，并没有捕获函数上下文的能力，因此无法直接指向成员函数；而函数对象可以通过std::bind绑定特定的参数或者具体的实例，并返回一个函数对象，因此可以指向成员函数。

用法： 
1. 普通函数：如果需要返回int类型输入一个参数： std::function<int(int)> func  和 int (*func)(int)
````c++
int foo(int a){

}
std::function<int(int)> func = &foo;
func(1);
int (*func)(int) = foo;
func(1);
````
2. `成员函数`：函数指针调用成员函数时必须依赖具体的对象，而function对象可以通过std::bind绑定成员函数与具体实例
````c++
// 成员函数
A aa;
// 1. 函数指针使用方法
int (A:: * p3)(int, int); // p3 是 A::* 成员函数类型
p3 = &A::a;
(aa.*p3)(3, 4);
// 2. function对象未绑定方法
function<int(A*, int, int)> p4 = &A::a;
p4(&aa, 3, 4);
// 3. 使用std::bind绑定使用
auto p5 = std::bind(&A::a, &aa,std::placeholders::_1, std::placeholders::_2);
p5(3, 4);
````

- std::function 比传统函数指针更灵活，支持以下特性：

| 特性             | std::function            | 函数指针                        |
|------------------|--------------------------|---------------------------------|
| 支持的调用对象   | 函数、Lambda、函数对象等 | 仅普通函数或静态成员            |
| 捕获上下文（闭包) | 支持（如 Lambda）        | 不支持                          |
| 类型安全         | 高                        | 低（易出现类型转换错误）        |
| 空值检查         | 支持（if (func) {...}）  | 需手动检查 nullptr              |


### sizeof与strlen的区别
1. sizeof 是在编译期就能确定的，可以计算类型或者变量的大小，底层根据具体的类型计算大小，所以时间复杂度为o(1)
2. strlen 只能计算char*指向内存中一块连续的char数组大小，在运行时动态计算，通过遍历的方式直到'/0'结束，时间复杂度为o(n)

### auto与decltype
- auto关键字自动类型推导在编译时确认，限制：1.定义时初始化 2.不能作为函数参数  3.不能作为类的非静态成员 4.不能定义数组 5.不会推导const、constexpr、static还有&修饰符

- decltype用于精确的类型推导，用法: decltype(exp),exp可以是变量名、表达式。会保存完整的修饰符const、static等。

# 用法
### 函数声明与定义分离
这样做的好处有三个：1.隐藏实现细节，避免暴露给用户 2.解耦性强，提高了灵活性 3.重用性高，不同地方实现符合自己需求的函数.
- 下面是一个样例，但是需要注意对于静态方法的外部定义不需要添加static
````c++
class HashBucket {
public: // 提供给外部的方法：初始化内存池、获取特定内存池、分配内存、释放内存
	static void initMemoryPool();
	static MemoryPool& getMemoryPool(int index);
	....
}
MemoryPool& HashBucket::getMemoryPool(int index) {
	static MemoryPool memoryPool[MEMORY_POOL_NUM]; // 静态局部变量从程序执行开始到结束
	return memoryPool[index];
}
````

### 左值右值、左右引用、移动语义、万能引用、完美转发

#### 左值右值
- 左值：一般来说左值指等号左边、可以取地址并有名字的就是左值 `int&& a = 1、字符串常量`是左值，编译器会优化创建一个临时的字符串变量
- 右值：等号右边的表达式 `x++`返回的是局部变量因此是右值

#### 左值引用右值引用
引用实际上就是对变量的别名，因此普通的左右值引用就是对左右值的引用，左值用&,右值用&&
- 右值->左值引用类型 使用const T& x = 右值，这是因为const在编译器优化一个常量，`因此形参为const T&类型可以接受右值引用，可以用来写移动拷贝构造函数`
- 左值->右值引用类型 `强制转换`: std::move\static_cast<T&&>\(T&&)

#### 移动语意
- 为了理解移动语意，需要先了解什么是浅拷贝与深拷贝
##### 浅拷贝、深拷贝
- 浅拷贝类似于引用传递，对于指针和对象来说，浅拷贝相当于指向同一块内存地址，这样做可能导致不安全问题。
- 深拷贝定义为创建一个与目标同大小的内存区域，并按照内容一一复制；一般拷贝构造、拷贝赋值使用深拷贝。深拷贝非常安全但是会带来额外开销
##### 移动语意定义
为了对于带有资源的对象，在拷贝完后不再使用的场景，移动语意实现资源转移且转移后能保证安全，确保被转移对象失去对资源的掌控，从而达到在实现浅拷贝的同时，得到深拷贝的效果。所以实现移动语意的关键在于右值，因为右值意味着表达式而且没有名字，因此触发移动构造，转移资源、减少拷贝

#### 万能引用
在了解万能引用之前必须知道引用折叠，引用折叠的目的是为了简化左值与右值之间的转换问题，只有当目标与当前值都是右值时，声明的变量才是右值类型，否则左值。
- 定义：万能引用就是为了实现一个函数模板，模板参数会动态的传入实参的左右值类型，动态声明形参的左右值引用类型。
- 存在问题：虽然形参声明为右值类型，但在方法体内实际运行过程中，形参有名字可取地址妥妥的一个左值；因此在后续调用其他方法使用形参时传入到其他方法是一个左值。

#### 完美转发
为了弥补万能引用中存在的从右值引用变到左值的问题，std::forward完美转发，能完美正确的保持原来的左右引用关系并向下层次转化，不再因右引用表达式是左值从而变掉


### 阻止对左值拷贝
通过设置基类的拷贝构造函数以及定义左值赋值运算符来阻止
- 目的是：基类可能管理不可复制的资源（如文件句柄、网络连接或unique_ptr），拷贝会导致资源重复释放或逻辑错误。
````c++
class nonecopyable{
	nonecopyable(const nonecopyable& other) = delete;
	nonecopyable& operator=(const nonecopyable& other) = delete;
}
````

### 跨源文件使用其他文件的类
- 方法一：使用头文件与源文件分离方法

具体： 类声明放入.h文件，类定义在.cpp文件；其他文件使用时只需导入.h文件就能在确保不会重复定义的情况下使用。
````c++
// ClassB.h
#pragma once  // 防止头文件重复包含
class ClassB {
public:
    void doSomething();  // 声明成员函数
};

// ClassB.cpp
#include "ClassB.h"
#include <iostream>

void ClassB::doSomething() {  // 定义成员函数
    std::cout << "ClassB is working!" << std::endl;
}

// ClassA.cpp
#include "ClassB.h"  // 包含 ClassB 的头文件

class ClassA {
public:
    void useClassB() {
        ClassB b;
        b.doSomething();  // 使用 ClassB
    }
};
````
- 方法二：前向声明 (解决头文件循环依赖)

具体：这种用法多在头文件中声明函数时需要使用到其他文件的类；这里是预先声明但无任何定义（所以不能实例化），然后让编译器在链接时去寻找具体定义。
````c++
// ClassA.h
#pragma once
class ClassB;  // 前向声明 ClassB

class ClassA {
public:
    void callClassB(ClassB& b);  // 使用 ClassB 的引用
};

// ClassB.h
#pragma once
class ClassA;  // 前向声明 ClassA

class ClassB {
public:
    void callClassA(ClassA& a);  // 使用 ClassA 的引用
};

// ClassA.cpp
#include "ClassA.h"
#include "ClassB.h"  // 实际调用时需要包含 ClassB.h

void ClassA::callClassB(ClassB& b) {
    // 可以操作 b 的成员函数
}
````
### 派生类继承基类时使用public与不使用public的区别
个人理解： 对于结构体而言继承默认使用public，而对于类继承默认private。那么就会导致基类中对于对象可见的成员方法或者成员变量变成private，`导致无法向上转型`。

- 使用场景：
- - public继承：表示“是一个（is-a）”关系，派生类是基类的子类型（如Dog继承Animal）。
- - private继承：表示“根据…实现（implemented-in-terms-of）”关系，仅`复用基类实现，不暴露接口（类似组合）`。

### 类和结构体对象内存大小计算
个人理解：
1. 空类对象的大小默认为1字节，是为了区分不同实例；如果被继承那么1字节会被优化0. 
2. 对于非空类，首先应该判断是否有`虚函数`如果有则根据操作系统位数增加字节。
3. 对象内存计算与成员变量的`顺序`和类内部所有变量（包括其他类对象）的`最大字节的基本数据类型`有关
- 不同的顺序导致不同的结果，这是因为：数据是根据自身的大小对齐
````c++
class A{
	char b;
	int a;
	char c;
}
A a; a的大小为12字节；

class B{
	char b;
	char c;
	int a;
}
B b; b的大小为8字节；
````
- 最大字节的基本数据类型： 1，4，8，1-》24字节
````c++
class A{
	char a;
	int b;
	long long c;
	char d;
}
A a; a的大小为24字节；

class B{
	A a;
	int b;
}
B b; b的大小为32字节，因为A中包含最大数据类型的8字节
````


### 虚表指针
1. 虚表指针的获取

只有当派生类或者当前类中存在虚函数才有虚表指针；当对象创建时，虚表指针会存放在对象地址的首部。
```` c++
class E {
public:
	virtual void foo() {

	};
};

class F :public E {
	
};

E e;
F f; // f与e的虚表地址一致
cout << **reinterpret_cast<void***>(&e)<<endl; // &e是一个指针， *void**指向的e的值（地址），**void*** 指向的首地址上值（64位8个字节,也就是虚表地址）
cout << **reinterpret_cast<void***>(&f);
````
2. 如何判断虚表指针指向

上面的例子就已经说明了：如果当前类存在虚函数则指向自身类模板中的虚表地址，如果自身不存在虚函数则与父类共享虚表。

### 模板特化
理解：模板特化分为全特化和偏特化，其中对于函数模板只存在全特化，函数模板的偏特化可以通过重载实现。模板特化在模板的基础上，通过修改不同的具体类型实现类似于重载的操作。
````c++
class A {
public:
	void a(T1 t1, T2 t2) {
		cout << sizeof(t1) << endl;
		cout << sizeof(t2) << endl;
	}
};

template<typename T2>
class A<char,T2> {
public:
	void a(T2 t2) {
		cout <<"偏特化：" << sizeof(t2) << endl;
	}
};

template<>
class A<char, int> {
public:
	void a() {
		cout << "全特化" << endl;
	}
};

template<>
class A<int , char> {
public:
	void a() {
		cout << "全特化1" << endl;
	}
};
````

### C++实现可变参数的三个方法
#### C方法 va_list 
用法： 导入<stdarg.h>,声明一个`va_list`变量x, 用`va_start(x,n)`方法将参数拷贝到x中，n是长度；可通过遍历`va_arg(x,T)`方式获取参数值，最后使用va_end结尾。
- 注意：要使用C方法变长参数，首先在函数声明里用...声明列表（必须在最后，且不能只有...）

````C++
#include<stdarg.h>
int f(int n,...){  // !! 这里的int n就决定了 可变参数...仅支持int类型 !!
	va_list arg;
	va_start(arg,10);

	int ans = 0;
	for(int i=0;i<n;i++)ans += va_arg(arg,int); // ！！ 由传入参数n的类型决定每次往后读取多少个字节大小
	va_end(arg); // 清理结尾
	return ans;
}
````

#### C++方法：使用initializer_list
initializer_list传参则要简单的多，首先这个东西自己就是一种模板容器，使用方法和vector类似。
````C++
#include <initializer_list>
 
int max(std::initializer_list<int> li) {
    int ans = 1 << 31;
    for (auto x: li) ans = ans>x ? ans : x;
    return ans;
}
 
main() {
	printf("%d\n", max({1, 2, 3})); //加上大括号，作为整体调用
}
````
- 缺点：只支持单一类型；只能读不能写；需要额外加括号，格式不统一

#### C++方法：可变参数模板
定义：一个可变参数模板（variadic template）就是一个接受可变数目参数的模板函数或模板类。 对于传递的可变参数不同，编译器会在编译期实例化所有的版本！！！
````C++
template<class T, class... Args> //Args：“模板参数包”
void foo(const T &t, const Args&... test); //test：“一个参数包（含有0或多个参数）”
 
foo(i, s, 42, d); //包中有三个参数
foo(s, 42, "hi"); //包中有两个参数
foo(d, s); //包中有一个参数
foo("hi"); //空包

// 编译器实例化
void foo(const int&,const string&,const int&,const double& );
void foo(const string&,const int&, const char [3]&);
void foo(const double&, const string&);
void foo(const char[3] &);
````

- 一般使用递归方式展开：
````c++
template<class T>
void print(T &t) {cout << t <<'\n';}
 
template<class T, class... Args>
void print(T &t, Args&... rest) {
    cout << t << ' ';
    print(rest...); // 打印剩余参数，注意省略号必须有
}
````
````c++
template<typename T, typename... Args>  // typename... Args可变参数模板；Args... args 接受多个参数；sizeof...(Args)获取参数个数；print(args...)展开参数包；std::forward<Args>(args)...完美转发
T* newElement(Args&&... args) {
	T* p = nullptr;
	if (p = reinterpret_cast<T*>(HashBucket::useMemory(sizeof(T))) != nullptr) {  // p是指向T大小的内存，并未初始化
		new(p) T(std::forward<Args>(args)...); // ？？这是什么操作？ new(p)表示在内存块p上构造T对象；std::forward<Args>(args)...完美转发当前函数的参数给T的构造函数
	}
	return p;
}
````

### enable_shared_from_this<> 是什么？以及为什么要用？
定义：是 C++ 标准库提供的一个工具类模板，用于解决 从对象内部安全获取指向自身的 std::shared_ptr 的问题。

问题场景：假如已有一个对象被共享指针管理，现在对象内部方法需要将this指针传递给其他需要std::shared_ptr的代码。如果单纯将this传入，那么会调用带参构造创建一个新的共享指针从而在析构对象时出现double-free错误。

解决思路：对于从被共享指针管理的对象获取到一个共享计数器的共享指针只能通过拷贝构造或拷贝赋值运算符获取。在enable_shared_from_this中声明了一个指向当前对象的weak_ptr指针，因此调用shared_from_this()时，会调用weak_ptr指针的.lock()方法返回一个共享指针对象。

