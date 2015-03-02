#More Effective C++笔记
------

必读经典

------
[TOC]

##Item 1：指针与引用的区别
二者之间的区别是：
>* 在任何情况下都不能用指向空值的引用，而指针则可以；
>* 指针可以被重新赋值以指向另一个不同的对象，但是引用则总是指向在初始化时被指定的对象，以后不能改变
if you did，what will happen？

```cpp
    char *p = NULL;
    char &rc = *p;//anything do to rc ,core
```
w
使用引用作为函数参数的好处是，不必测试他是否为空。
在以下情况下使用指针：
>* 一是存在不指向任何对象的可能性
>* 二是需要能够在不同的时刻指向不同的对象
在以下情况使用引用：总是指向一个对象且一旦指向一个对象之后就不会改变指向；重载某个操作符时，使用指针会造成语义误解。

```cpp
vector<int> a(10);
a[4] = 10; // 正常语法
*a[4] = 10;// 诡异的语法，所以当重载才操作符时，应该返回reference
```

##Item 2：尽量使用C＋＋风格的类型转换
static_cast：功能上基本上与C风格的类型转换一样强大，含义也一样，**它不能从表达式中去除const属性。**
```cpp
int firstNumber,secondNumber;
double ret = ( (double)firstNumber)/secondNumber;//旧式转换
double ret = static_cast<double>(firstNumber)/secondNumber;
```
此外，C风格转换允许将struct转换为int,double转换为指针，static_cast不行。

const_cast：用于类型转换掉表达式的const或volatileness属性但是不能用它来完成修改这两个属性之外的事情，而所谓constness与volatileness是被编译器约束的。
```cpp
void update(widget& c)
{
}
int main(int argc, char *argv[])
{
    widget  w;
    const widget& rw = w;
    update(w);
//  update(rw));// 错误
    update(const_cast<widget&>(rw));
}
```
dynamic_cast：将指向基类的指针或者引用转换为派生或者兄弟类的指针或者reference ，返回null指针（转型对象是指针）或exception（转型对象是reference）；
```cpp
#include <iostream>
#include <stdio.h>
#include <string>
class BaseClass
{
public:
    int m_iNum;
    virtual void foo(){}; //基类必须有虚函数。保持多态特性才能使用dynamic_cast
};

class DerivedClass: public BaseClass
{
public:
    char*m_szName[100];
    void bar(){};
};

int main(int argc, char *argv[])
{

    BaseClass* pDerived =new DerivedClass();
    DerivedClass *pd1 = static_cast<DerivedClass *>(pDerived); //子类->父类，静态类型转换，正确但不推荐
    DerivedClass *pd2 = dynamic_cast<DerivedClass *>(pDerived); //子类->父类，动态类型转换，正确

    BaseClass* pBase =new BaseClass();
    DerivedClass *pd21 = static_cast<DerivedClass *>(pBase); //父类->子类，静态类型转换，危险！访问子类m_szName成员越界    
    DerivedClass *pd22 = dynamic_cast<DerivedClass *>(pBase); //父类->子类，动态类型转换，安全的。结果是NULL
    return 0;
}
```
reinterpret_cast：最普通的用途是在函数指针类型的转换，**使用它的代码很难移植**，因为他的转换结果是在执行期定义，少用。

##Item 3：不要使用多态性数组
继承的重要特征是通过指向其基类对象的指针或引用，来操作继承基类的对象。多态和指针算法不能混合在一起使用，所以数组和多态也不能用在一起
数组中各元素的内存地址是数组的起始地址加上之前各个元素的大小得到的，如果各元素大小不一，那么编译器将不能正确地定位元素，从而产生错误.

##Item 4：避免无用的缺省构造函数
缺省构造函数则可以不利用任何在建立对象时的外部数据就能初始
化对象。
没有缺省构造函数造成的问题：通常不可能建立对象数组，对于使用非堆数组，可以在定义时提供必要的参数，
```cpp
class Equipment
{
public:
	Equipment(int IDNumber);
};
Equipment bestE[10];//错误
Equipment *bestE = new Equipment[10];//没有正确调用构造函数
//构建非堆数组方法:
Equipment bestE[] = {
Equipment(1),
Equipment(2),
...
};
```
另一种方法是使用指针数组，但是必须删除数组里的每个指针指向的对象，而且还增加了内存分配量。
```cpp
void *rawMemory = operator new[](10*sizeof(EquipmentPiece)); 
// make bestPieces point to it so it can be treated as an 
// EquipmentPiece array 
EquipmentPiece *bestPieces =static_cast<EquipmentPiece*>(rawMemory); 
// construct the EquipmentPiece objects in the memory 
// 使用"placement new" (参见条款M8) 
for (int i = 0; i < 10; ++i) 
{
	  new (&bestPieces[i]) EquipmentPiece( ID Number ); 
}

//析构他们
for (int i = 9; i >= 0; --i)
{
  bestPieces[i].~EquipmentPiece();
}
// deallocate the raw memory 
operator delete[](rawMemory); 
```

##Item 5：谨慎定义类型转换函数
关于类型转换的小复习
```cpp
class Rational
{
	operator double() const;// 不好的做法，将Rational转换为double的函数
	double asDouble() const;//正确做法
};

Rational r(1,2);
double ret = 1.3*r;//自动转换r为double
cout<< r; // 将自动转换为double ，这不是我们要的
cout<< r.asDouble();//正确做法
```
如上例子，缺省的隐式转换将带来出乎意料的结果，因此应该尽量消除，使用显式转换函数通过不声明运算符的方法，可以克服隐式类型转换运算符的缺点。
通过使用explicit关键字和**代理类(纳尼？后续Item 30会讲到)**的方法可以消除单参数构造函数造成的隐式转换
来个更险恶的隐式转换例子：
```
class Array
{
public:
	Array(int);
	bool operator==( const Array<int>& lhs,const Array<int>& rhs);
};

//问题来了
Array a(10),b(10);
for(int i=0;i<10;++i)
{	if(a == b[i]) //手误写错a[i]为a , 真是非常险恶的bug！！
	{
	}else
	{
	}
}
```
上述例子中，编译器发现a是数组，b却是一个int，但a.operator==(b)的第二个参数其实是Array类型，**编译器接着发现Array有个用int就能构造的对象，遂隐式转换之**，bug~

##Item 6：自增和自减操作符前缀形式与后缀形式的区别
大师说，当年没有办法区分++和--操作符的前缀与后缀调用，大家纷纷抱怨，才有了今天的支持两种形式的重载。
```
class UPInt{
	UPInt& operator++();//++前缀
	const UPInt operator++(int);//后缀++
};
```
后缀式返回const对象，原因是 ：使该类的行为和int一致，而int不允许连续两次自增后缀运算；连续两次运算实际只增一次，和直觉不符。
```
UPInt p;
p++++;// 不科学，遂通过返回const 禁止之
//因为p.operator++(0).operator++(0);是错误的
```
一个const对象调用的operator++是一个no-const成员函数，所以，const对象不能调用他，我们的目的达到了。
前缀比后缀效率更高，因为后缀要返回对象，而前缀只返回引用另外，可以用前缀来实现后缀，以方便维护

##Item 7：不要重载&&，||，或者，
对于以上操作符来说，计算的顺序是从左到右，返回最右边表达式的值如果重载的话，不能保证其计算顺序和基本类型想同操作符重载的目的是使程序更容易阅 读，书写和理解，而不是来迷惑其他人如果没有一个好理由重载操作符，就不要重载而对于&&，||和，，很难找到一个好理由

##Item 8：理解各种不同含义的new和delete
new操作符完成的功能分两部分：第一部分是分配足够的内存以便容纳所需类型的对象；第二部分是它调用构造函数初始化内存中的对象new操作符总是做这两件事，我们不能以任何方式改变它的行为
我们能改变的是如何为对象分配内存new操作符通过调用operator new来完成必需的内存分配，可以重写或重载这个函数来改变它的行为可以显式调用operator来分配原始内存
如果已经分配了内存，需要以此内存来构造对象，可以使用placement new，其调用形式为new(void* buffer)class(int size)
```
//placement的例子
class Widget { 
public: 
  Widget(int widgetSize); 
  ... 
}; 
Widget * constructWidgetInBuffer(void *buffer, 
                                 int widgetSize) 
{ 
  return new (buffer) Widget(widgetSize); 
} 
```
对于delete来说，应该和new保持一致，怎样分配内存，就应该采用相应的办法释放内存
operator new[]与operator delete[]和new与delete相类似

##Item 9：使用析构函数防止资源泄漏
使用指针时，如果在delete指针之前产生异常，将会导致不能删除指针，从而产生资源泄漏
解决办法：使用对象封装资源，如使用auto_ptr，使得资源能够自动被释放


##Item 10：在构造函数中防止资源泄漏
类中存在指针时，在构造函数中需要考虑出现异常的情况：异常将导致以前初始化的其它指针成员不能删除，从而产生资源泄漏解决办法是在构造函数中考虑异常处理，产生异常时释放已分配的资源最好的方法是使用对象封装资源

##Item 11：禁止异常信息传递到析构函数外
禁止异常传递到析构函数外的两个原因：
>* 第一能够在异常传递的堆栈辗转开解的过程中，防止terminate被调用；
>* 第二它能帮助确保析构函数总能完成我们希望它做的所有事情

解决方法是在析构函数中使用try-catch块屏蔽所有异常

##Item 12：理解抛出一个异常与传递一个参数或调用一个虚函数间的差异
异常有这么多新概念闻所未闻，真是大开眼界，有人说C++的异常是一个鸡肋，但大师说异常是很有用的，当年我们喜欢用errno去detect错误，
```cpp
void f(Widget w) ...                 // 通过传值
catch (Widget w) ...                 // 通过传值捕获异常 
catch (Widget& w) ...               // 通过传递引用捕获 
catch (const Widget& w) ...        // 通过传递指向const的引用
```
传递函数参数与异常的途径可以是传值、传递引用或传递指针，这是相同点，但系统完成的操作过程是完全不同的，产生的原因是：你调用函数时，程序的控制权最终还会返回到函数的调用处，但是
当你抛出一个异常时，控制权永远不会回到抛出异常的地方。**所以无论通过传值还是引用传递异常的参数都会由于离开生存空间析构掉，他所以必须传递的是拷贝。**，另外，被拷贝的是静态类型而非动态。
```cpp
class Widget { ... }; 
class SpecialWidget: public Widget { ... }; 
void passAndThrowWidget() 
{ 
  SpecialWidget localSpecialWidget; 
  ... 
  Widget& rw = localSpecialWidget;      // rw 引用SpecialWidget 
  throw rw;                            // 它抛出一个类型为Widget的异常 
} 
```
```
catch (Widget& w)                 // 捕获Widget异常 
{ 
  throw;                          // 重新抛出异常，让它继续传递，good，
								  //他仍是SpecialWidget的异常
}
catch (Widget& w)               
{ 
  throw w;                        // 传递被捕获异常的拷贝，bad，改变了w的类型
} 
```
有三个主要区别：
>* 第一，异常对象在传递时总被进行拷贝，当通过传值方式捕获时，**异常对象被拷贝了两次对象**，作为参数传递给函数时不一定需要被拷贝；
>* 第二，对象作为异常被抛出与作为参数传递给函数相比，前者类型转换比后者少（前者只有两种转换形式：继承类与基类的转换，类型化指针到无类型指针的转换）；
>* 最后一点， catch子句进行异常类型匹配的顺序是它们在源代码中出现的顺序，第一个类型匹配成功的擦他处将被用来执行当一个对象调用一个虚函数时，被选择的函数 位于与对象类型匹配最佳的类里，急事该类不是在源代码的最前头

##Item 13：通过引用捕获异常
有三个选择可以捕获异常：第一指 针，建立在堆中的对象必需删除，而对于不是建立在堆中的对象，删除它会造成不可预测的后果，因此将面临一个难题：对象建立在堆中还是不在堆中；第二传 值，异常对象被抛出时系统将对异常对象拷贝两次，而且它会产生对象切割，即派生类的异常对象被作为基类异常对象捕获时，它的派生类行为就被切割调了 这样产生的对象实际上是基类对象；第三引用，完美解决以上问题

##Item 14：审慎使用异常规格
避免调用unexpected函数 的办法：第一避免在带有类型参数的模板内使用异常规格因为我们没有办法知道某种模板类型参数抛出什么样的异常，所以不可能为一个模板提供一个有意义的 异常规格；第二如果在一个函数内调用其它没有异常规格的函数时应该去除这个函数的异常规格；第三处理系统本身抛出的异常可以将所有的 unexpected异常都被替换为自定义的异常对象，或者替换unexpected函数，使其重新抛出当前异常，这样异常将被替换为 bad_exception，从而代替原来的异常继续传递
很容易写出违反异常规格的代码，所以应该审慎使用异常规格

##Item 15：了解异常处理的系统开销
三 个方面：第一需要空间建立数据结构来跟踪对象是否被完全构造，还需要系统时间保持这些数据结构不断更新；第二try块无论何时使用它，都得为此付出 代价编译器为异常规格生成的代码与它们为try块生成的代码一样多，所以一个异常规格一般花掉与try块一样多的系统开销第三抛出异常的开销因为 异常很少见，所以这样的事件不会对整个程序的性能造成太大的影响

##Item 16：牢记80%/20%准则
80-20准则说的是大约20%的代码使用了80%的程序资源，即软件整体的性能取决于代码组成中的一小部分使用profiler来确定程序中的那20%，关注那些局部效率能够被极大提高的地方。
大师告诫我们，找瓶颈不是靠猜，大多数程序员们的直觉是错的，程序的性能特征往往不能靠直觉来确定，真确的发生在我自己和周围的“为了提高程序各部分效率而倾注了大量精力，但却吃力不讨好，对整体的程序性能没有显著影响”。
我们应该使用profiler去识别程序的20%部分，当然profiler如果不输入有代表性的数据，其显示的结果不一定有帮助，这是一个巨大的话题，将会晚点讨论。

##Item 17：考虑使用懒惰计算法
懒惰计算法的含义是**拖延计算的时间**，等到需要时才进行计算。
>* 引用计数，其实是在说string的copy on write
>* 还是在说string的读和写，栗子是：
```
String s = "Homer's Iliad";            // 假设是一个reference-counted string 
cout << s[3];                         // 调用 operator[] 读取s[3] 
s[3] = 'x';                           // 调用 operator[] 写入 s[3] 
```
如何在operator[]中判断是读取还是写入操作呢。抱歉做不到，但是**通过使用lazy evaluation和条款M30中讲述的proxy class，我们可以推迟做出是读操作还是写操作的决定，直到我们能判断出正确的答案。**
>* Lazy Fetching，是说对于大型对象，当 用到的时再从db、磁盘取
```
class LargeObject { 
public: 
  LargeObject(ObjectID id);  
  const string& field1() const; 
  int field2() const;
  ...  
private: 
  ObjectID oid;  
  mutable string *field1Value;// mutable声明以在const成员函数内修改他
  mutable int *field2Value;  
  ...  
};  
LargeObject::LargeObject(ObjectID id) 
: oid(id), field1Value(0), field2Value(0), field3Value(0), ... 
{}  
const string& LargeObject::field1() const 
{ 
  if (field1Value == 0) { 
   // 从数据库中为filed 1读取数据，使field1Value 指向这个值; 
}  
  return *field1Value; 
```
 

##Item 18：分期摊还期望的计算
核心是使用过度热情算法，有两种方法：
>* **缓存**那些已经被计算出来而以后还有可能需要的值；
>* **预提取**，做比当前需要做的更多事情。

当必须支持某些操作而不总需要其结果时，可以使用懒惰计算法提高程序运行效率；当必须支持某些操作而其结果几乎总是被需要或不止一次地需要时，可以使用过度热情算法提高程序运行效率

##Item 19：理解临时对象的来源
大师说往往有些人把临时对象和局部变量混淆，临时变量根本不出现在源代码中，是一个非堆对象。
临时对象产生的两种条件：**函数成功调用而进行隐式类型转换和函数返回对象时**。
```cpp
// 一个函数调用隐式转换产生临时对象的例子
void uppercasify(string& str);
char str[]="xxxxx";
uppercasify(str);//错误，因为str是一个reference to non const，c++不允许对其发生隐式转换（符合直觉）
//一个函数返回对象时产生临时对象的例子
const Number operator+(const Number& lhs,const Number& rhs); 
//改进方案在Item 22有讨论
```

临时对象是有开销的，因此要尽可能去消除它们，然而更重要的是训练自己寻找可能建立临时对象的地方，在任何时候只要见到常量引用参数，就存在建立临时对象而绑定在参数上的可能性。在任何时候只要见到函数返回对象，就会有一个临时对象被建立（以后被释放）。

##Item 20：协助完成返回值优化(RVO)
应当返回一个对象时不要试图返回一个指针或引用，炒冷饭了。总的来说：
```
const Rational operator*(const Rational& lhs,const Rational& rhs); 
//甚至不用看operator*的代码，我们就知道它肯定要返回一个对象
```
C+ +规则允许编译器优化不出现的临时对象，所有最佳的办法莫过于：
```
retrun Ratinal(lhs.numerator()*rhs.numerator(), lhs.denominator()*rhs.denominator())
```
这种优化是通过使用函数的retuan location（或者用在一个函数调用位置的对象来替代），来消除局部临时对象，这种优化还有一个名字：返回值优化

##Item 21：通过重载避免隐式类型转换
隐式类型转换将产生临时对象，从而带来额外的系统开销。
解决办法是使用重载，以避免隐式类型转换要注意的一点是在C++中有一条规则是**每一个重载的operator必须带有一个用户定义类型的参数**（这条规定是有道理的，如果没有的话，程序员将能改变预定义的操作，这样做肯定把程序引入混乱的境地，10+10都的预定义都被改变啦）
另外，牢记8020规则，没有必要实现大量的重载函数，除非有理由确信程序使用重载函数后整体效率会有显著提高

##Item 22：考虑用运算符的赋值形式取代其单独形式
运算符的赋值形式不需要产生临时对象，因此应该尽量使用对运算符的单独形式的最佳实现方法
```
return Rational(lhs) += rhs;
```
这种方法将返回值优化和运算符的赋值形式结合起来，即高效，又方便。
##Item 23：考虑变更程序库
程序库必须在效率和功能等各个方面有各自的权衡，因此在具体实现时应该考虑利用程序库的优点例如程序存在I/O瓶颈，就可以考虑用stdio替代iostream。

##Item 24：理解虚拟函数多继承虚基类和RTTI所需的代价
虚函数所需的代价：必须为每个包含虚函数的类的virtual table留出空间；每个包含虚函数的类的对象里，必须为额外的指针付出代价；
```
void makeACall(C1 *pC1) 
{ 
  pC1->f1();//这种调用，虚函数本身不是性能瓶颈，虚函数所需要的代价与内连有关
} 
```
虚函数是不能内联的？这是因为“内联”是指“在编译期间用被调用的函数体本身来代替函数调用的指令，”但是虚函数的“虚”是指“直到运行时才能知道要调用的是哪一个函数。

多继承时，在单个对象里有多个vptr（一个基类对应一个）它和虚基类一样，会增加对象体积的大小。
RTTI能让我们在运行时找到对象和类的有关信息，所以肯定有某个地方存储了这些信息，让我们查询这些信息被存储在类型为type_info的对象里，可以通过typeid操作符访问到一个类的typeid对象通常，RTTI被设计为在类的vbtl上实现。

##Item 25：将构造函数和非成员函数虚拟化
在类中构建一个虚拟函数，其功能仅仅是实现构造函数，就可以对外界提供**一组派生类的公共构造接口**，这就是虚拟构造函数了。
```
class NLComponent {               //用于 newsletter components  
public:                           // 的抽象基类  
  ...                             //包含至少一个纯虚函数 
}; 
class TextBlock: public NLComponent { 
public: 
  ...                             // 不包含纯虚函数 
};  
class Graphic: public NLComponent { 
public: 
  ...                             // 不包含纯虚函数 
}; 

class NewsLetter
{ 
public: 
  ...  
private: 
  // 为建立下一个NLComponent对象从str读取数据, 
  // 建立component 并返回一个指针。 
  static NLComponent * readComponent(istream& str); 
   ... 
}; 
  NewsLetter::NewsLetter(istream& str) // 这就是虚拟构造函数了
  { 
    while (str) { 
        components.push_back(readComponent(str)); //根据输入的数据不同，建立的不同的派生类对象
    } 
 } 
```

虚拟拷贝构造函数也是可以实现的，但是要利用到最近才被采纳的较宽松的虚拟函数返回值类型规则，被派生类重定义的虚拟函数不用必须与基类的虚拟函数具有一样的返回类型。
```	
class NLComponent { 
public: 
  // declaration of virtual copy constructor 
  virtual NLComponent * clone() const = 0; 
  ...  
 }
class TextBlock: public NLComponent { 
public: 
  virtual TextBlock * clone() const         // virtual copy 
  { return new TextBlock(*this); }          // constructor 
  ...  
};  
class Graphic: public NLComponent { 
public: 
  virtual Graphic * clone() const            // virtual copy constructor ，返回值不一定要和基类的类型相同
  { return new Graphic(*this); }             
  ...  
}; 
//有了虚拟拷贝构造函数，新派生类的拷贝构造变得很容易了
NewsLetter::NewsLetter(const NewsLetter& rhs) 
{
	__foreach(rhs,it)
	{  
		components.push_back((*it)->clone()); //
	} 
} 
```
具有虚拟行为的非成员函数很简单首先编写一个虚拟函数完成工作，然后再写一个非虚拟函数，它什么也不做只是调用这个函数，可以使用内联来避免函数调用的开销.
```
class NLComponent { 
public: 
  virtual ostream& print(ostream& s) const = 0; 
  ...  
};  
class TextBlock: public NLComponent { 
public: 
  virtual ostream& print(ostream& s) const; 
  ...  
};  
class Graphic: public NLComponent { 
public: 
  virtual ostream& print(ostream& s) const; 
  ...  
};  
inline ostream& operator<<(ostream& s, const NLComponent& c) 
{ 
  return c.print(s); 
} 
//做的好，现在可以正常语法调用他们了,如果他是成员函数，得反过来，g<< cout;
TextBlock t;
Graphic g;
cout << t << g;

```

##Item 26：限制某个类所能产生的对象数量
只有一个对象：使用单一模式，将类的构造函数声明为private，再声明一个静态函数，该函数中有一个类的静态对象，不将该静态对象放在类中原因是放在函数中时，执行函数时才建立对象，并且对象初始化时间确定的，即第一次执行该函数时，另外，**该函数不能声明为内联，如果内联可能造成程序的静态对象拷贝**。(大师多次，多次提到不同编译单元，translation unit（也就是生成一个object文件的源代码的集合）内的静态成员被初始化顺序是不确定的)。
```
class Printer { 
public: 
  static Printer& thePrinter(); 
  ... 
private: 
  Printer(); 
  Printer(const Printer& rhs); 
  ...  
}; 
Printer& Printer::thePrinter() //单例模式惯用法，被称之为Meyers' Singleton ？
{ 
  static Printer p; 
  return p; 
} 
```
超过一个限制对象个数：建立一个基类，构造函数中计数加一，若超过最大值则抛出异常；析构函数中计数减一。
```
#include <iostream>
#include<exception>
using namespace std;
template<class BeingCounted> 
class Counted { 
public: 
  class TooManyObjects:public exception{};                     // 用来抛出异常  
  static int objectCount() { return numObjects; }  
protected: 
  Counted(); 
  Counted(const Counted& rhs);  
  ~Counted() { --numObjects; }  
private: 
  static int numObjects; 
  static const int maxObjects;  
  void init();                                // 避免构造函数的 
};

template<class BeingCounted> 
Counted<BeingCounted>::Counted() 
{ init(); }  
template<class BeingCounted> 
Counted<BeingCounted>::Counted(const Counted<BeingCounted>&) 
{ init(); }  
template<class BeingCounted> 
void Counted<BeingCounted>::init() 
{ 
  if (numObjects >= maxObjects) throw TooManyObjects(); 
  ++numObjects;
}

class Printer: private Counted<Printer> { 
public: 
  // 伪构造函数 
  static Printer * makePrinter()
  {
  	return new Printer;
  }
  static Printer * makePrinter(const Printer& rhs)
  {
  	return new Printer(rhs);
  }
  ~Printer();  
  // private继承，这里我们必须使用using来恢复public访问，当然不是必须，如果你没用到的话
  using Counted<Printer>::objectCount;      
  using Counted<Printer>::TooManyObjects;  
private: 
  Printer(){}; 
  Printer(const Printer& rhs){}; 
};
      
const int Counted<Printer>::maxObjects = 1; 
int Counted<Printer>::numObjects; //自动初始化为0

int main()
{
	try
	{
	   Printer * p = Printer::makePrinter();
	   Printer * p2 = Printer::makePrinter();   
	}catch(std::exception& e)
	{
		cout<<e.what();
	}
	
	return 0;
}

```


##Item 27：要求或禁止在堆中产生对象
```
//只能new的对象
class ObjNewOnly
{
public:
	ObjNewOnly（）{};
	void destory() const //需要帮忙delete this
	{
		delete this;		
	}
private:// 应该改为protected:成员才能fix继承和组合问题
	~ObjNewOnly()
	{
		cout << "destorying"<<endl;
	}
};
//当你需要继承，或者组合时，遇到了困难，使用protected来解决继承问题，
// protected成员，派生类可以访问 b
int main(int argc, char *argv[])
{
	ObjNewOnly *po = new ObjNewOnly;
	po->destory();
//	delete po;// error 
	return 0;
}
```
没有办法不能判断一个对象是否在堆中，但是可以判断一个对象是否可以安全用delete删除，只需在operator new中将其指针加入一个列表，然后根据此列表进行判断。
把一个指针dynamic_cast成void*类型（或const void*或volatile void*等），生成的指针将指向原指针指向对象内存的开始处，但是dynamic_cast只能用于指向至少具有一个虚拟函数的对象的指针上。有点走火入魔了啊。。。。
禁止建立堆对象可以简单的将operator new声明为private，但是仍然不能判断其是否在堆中

##Item 28：灵巧（smart）指针
（note，晚点回来细读）
灵巧指针的用处是可以对操作进行封装，统一用户接口。
灵巧指针从模板生成，因为要与内建指针类似，必须是强类型的；模板参数确定指向对象的类型。
灵巧指针的拷贝和赋值，采取的方案是当auto_ptr被拷贝和赋值时，对象所有权随之被传递此时，通过传值方式传递灵巧指针对象将导致不确定的后果，应该使用引用。
记住当返回类型是基类而返回对象实际上派生类对象时，不能传递对象，应该传递引用或指针，否则将产生对象切割。
测试灵巧指针是否为NULL有两种方案：一种是使用类型转换，将其转换为void*，但是这样将导致类型不安全，因为不同类型的灵巧指针之间将能够互相比较；另一种是重载operator!，这种方案只能使用!ptr这种方式检测
最好不要提供转换到内建指针的隐式类型转换操作符，直接提供内建指针将破坏灵巧指针的灵巧特性
灵巧指针的继承类到基类的类型转换的一个最佳解决方案是使用模板成员函数，这将使得内建指针所有可以转换的类型也可以在灵巧指针中进行转换但是对于间接继承的情况，必须用dynamic_cast指定其要转换的类型是直接基类还是间接基类
为了实现const灵巧指针，可以新建一个类，该类从非const灵巧指针继承这样的化，const灵巧指针能做的，非const灵巧指针也能做，从而与标准形式相同

##Item 29：引用计数
不错的string refcount实现。
使用引用计数后，对象自己拥有自己，当没有人再使用它时，它自己自动销毁自己因此，引用计数是个简单的垃圾回收体系。
在基类中调用delete this将导致派生类的对象被销毁
写时拷贝：与其它对象共享一个值直到写操作时才拥有自己的拷贝它是Lazy原则的特例
精彩的类层次结构：
RCObject类提供计数操作；StringValue包含指向数据的指针并继承RCObject的计数操作；RCPtr是一个灵巧指针，封装了本属于String的一些计数操作。

##Item 30：代理类
可以用两个类来实现二维数组：Array1D是一个一维数组，而Array2D则是一个Array1D的一维数组Array1D的实例扮演的是一个在概念上不存在的一维数组，它是一个代理类
代 理类最神奇的功能是区分通过operator[]进行的是读操作还是写操作，它的思想是对于operator[]操作，返回的不是真正的对象，而是一个 proxy类，这个代理类记录了对象的信息，将它作为赋值操作的目标时，proxy类扮演的是左值，用其它方式使用它，proxy类扮演的是右值用赋值 操作符来实现左值操作，用隐式类型转换来实现右值操作
用proxy类区分operator[]作左值还是右值的局限性：要实现proxy类和原类型的无缝替代，必须申明原类型的一整套操作符；另外，使用proxy类还有隐式类型转换的所有缺点
编程点滴：不能将临时对象绑定为非const的引用的行参

##Item 31：让函数根据一个以上的对象来决定怎么虚拟
有 三种方式：用虚函数加RTTI，在派生类的重载虚函数中使用if-else对传进的不同类型参数执行不同的操作，这样做几乎放弃了封装，每增加一个新的类 型时，必须更新每一个基于RTTI的if-else链以处理这个新的类型，因此程序本质上是没有可维护性的；只使用虚函数，通过几次单独的虚函数调用，第 一次决定第一个对象的动态类型，第二次决定第二个对象动态类型，如此这般然而，这种方法的缺陷仍然是：每个类必须知道它的所有同胞类，增加新类时，所有 代码必须更新；模拟虚函数表，在类外建立一张模拟虚函数表，该表是类型和函数指针的映射，加入新类型是不须改动其它类代码，只需在类外增加一个处理函数即可

##Item 32：在未来时态开发程序
未来时态的考虑只是简单地增加了一些额外约束：
提供完备的类，即使某些部分现在还没有被使用
将接口设计得便于常见操作并防止常见错误使得类容易正确使用而不易用错
如果没有限制不能通用化代码，那么通用化它

##Item 33：将非尾端类设计为抽象类
如果有一个实体类公有继承自另一个实体类，应该将两个类的继承层次改为三个类的继承层次，通过创造一个新的抽象类并将其它两个实体类都从它继承因此，设计类层次的一般规则是：非尾端类应该是抽象类在处理外来的类库，可能不得不违反这个规则
编程点滴：抽象类的派生类不能是抽象类；实现纯虚函数一般不常见，但对纯虚析构函数，它必须实现

##Item 34：如何在同一程序中混合使用C++和C
混合编程的指导原则：
确保C++和C编译器产生兼容的obj文件
将在两种语言下都使用的函数申明为extern C
只要可能，用C++写main()
总用delete释放new分配的内存；总用free释放malloc分配的内存
将在两种语言间传递的东西限制在用C编译的数据结构的范围内；这些结构的C++版本可以包含非虚成员函数

##Item 35：让自己习惯使用标准C++语言
STL 基于三个基本概念：包容器（container）选择子（iterator）和算法（algorithms）包容器是被包容的对象的封装；选择子是类 指针的对象，让你能如同使用指针操作内建类型的数组一样操作STL的包容器；算法是对包容器进行处理的函数，并使用选择子来实现
