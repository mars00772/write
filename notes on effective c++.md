#Notes On Effective C++
------

据说应该禁止没有读过effective c++的程序员写c++。 但大师的文字幽默搞怪，遂读以志之。

------
[TOC]

##Item 1 多把C++当C++来用啊
##Item 2 prefer const enum inline to #defines
>* #define 没有放进symbol table，调试跟踪困难
	>1. 曾和构建工程师聊过，甚至public libs'version 放进 symbol tables也是有益的
	>2. use strings binfile helps to get symbols
>* #define 有作用域污染
	>1. 重构、svn整理噩梦

##Item 3 const
>* const 出现在*左边，表示被指物是常量,const char* p = xx,const data
>* 也有人写成char const * p xxx;
>* const出现在*右边，指针是常量，char *const p=xx,const p
>* 少用 const_iterator，作者另一本书说的
>* const 修饰函数
	>1. const X operator *（....) 返回const的必要性，是防止有人 if((a*b)=c) 手误！！
>* muteable修饰变量，const函数也可以修改

##Item 4 对象用前的初始化
>* 内建数据类型需要亲自初始化
>* 将非局部静态变量变为局部静态变量（放到函数内，返回这个对象的引用）来顺序初始化它们

##Item 5&6 构造函数etc
>* 空class的默认函数
	>1. 默认构造、析构、拷贝构造，operator=函数
>* 使用noncopyable来禁止拷贝
	>1.boost内有实现
	>2.不一定要以public继承，见[item32](#jump32),[item39](#jump39)
	>3.friend函数也没辙

```c++
class noncopyable
{
protected://派生类可以析构
	noncopyable() {}
	~noncopyable() {}
private:// 禁止拷贝和赋值
	noncopyable( const noncopyable& );
	const noncopyable& operator=( const noncopyable& );
};
class Test:public nocopyable//不一定以public继承
{
};
void func()
{
	Test a,b;
	Test c(a);//禁用拷贝构造函数
	b = a;//赋值函数
}

```
##<span id="jump7">Item 7</span> virtual 析构函数
>* C++明确指出当派生对象由基类指针删除，而该基类的析构函数不是virutal，其结果未定义。（局部销毁）
>* 反过来说，virtual会膨胀类对象大小
>* 像std::string、vector、list、setunordered_map标准库不带virtual析构函数，勿随便继承
>* 纯虚的析构函数？需要为其实现一个空的函数
>* 析构函数的运行方式当然是先析构派生类（否则怎么会有item7的故事呢），编译器会在派生类的析构函数上创建一个对基类析构函数~xx的调用，所以得实现
##Item 8 析构函数不要抛异常
>* 析构函数里面catch住，用std::abort死的迅速（不可接受)
>* 吞下异常，并成员变量记录状态，如bool bclosed =true;
>* dbcon这种close在析构里面做会有异常，解法是让客户端接手异常，不要在析构函数里面执行，明确提供一个普通函数来执行该操作
##Item 9 不要在构造和析构函数里面调用virtual函数
>* 派生类的构造顺序是，开始构造父类部分(step 1)-> 完成父类->构造子类->完成子类构造
>* 父类构造期间，virtual函数不是virutal函数，因为派生类成员未初始化，所以不会把virtual函数下降至派生类层次
>* 构造期间(step 1)，派生类不是派生类，而是父类对象
>* 析构期间，一旦派生类析构函数开始执行，其成员变量become未定义，进入父类析构函数后，这个对象就被看作是父类对象了
```c++
class base{
public:
	base(){
		//log();//这里调用的是自己的log函数,但编译不过 
		fakelog();//哈哈哈，骗过去了。。。但运行时错误
	}
	void fakelog(){
		log();
	}
	virtual void log() const = 0;//编译不过
};
class derived:public base {
public:
	virtual void log() const {}
};

void func(){
	derived d;
}
```

##Item 10 operator=返回一个this指针引用
##Item 11 operator=处理自我赋值
>* 问题的引发在于两个指向相同heap对象的指针在使用operator=函数时删了自己又赋值给自己的bug

```
class fk
{
public:
	fk& operator=(const fk& rhs){
		delete m_pb;
		m_pb = new fk_base(&rhs.m_pb);
		return *this;
	}
private:
	fk_base* m_pb;
};
```
##Item 12 Copy all parts of an object
>* 对于复杂对象，如成员变量有指向开辟好的内存的指针，默认拷贝函数的拷贝就不够聪明了
>* 于是需要自定义拷贝函数，但总有几点容易被忽视
	>1. 添加成员变量后，忘记更新拷贝函数（编译器不会发出警告），所以自己应该小心
	>2. 忘记拷贝父类的成员变量，办法是调用父类的拷贝函数，上代码

```cpp
class base{
public:
	base(const base& rhs){
	}
	base& operator=(const base& rhs);
private:
	string xxx;
};
class derived:public base{
public:
	derived(const derived& rhs):base(rhs){//调用父类构造函数
	}
	derived& operator=(const derived& rhs){
		base::operator=(rhs);//调用父类赋值，语法好。。怪
		xxx2=rhs.xxx2;
		return *this;
	}
private:
	string xxx2;
};
```
##Item 13 以对象管理资源
>* 常有人写出奇怪的代码：
```c++
class base{
....
};
void func(){
	base* pb = createBase();
	....// bug 过早return ，异常、导致delete不会被调用
	delete pb;
}
```
>* 应该将释放资源的代码放在对象的析构函数中，以保证资源被完整释放
>* 用share_ptr，注意其用的是delete而不是`delete[]`，所以`share_ptr<int>spInt(new int[32])`是馊主意，动态数组，请用vector、string代替吧
>* 上经典的mutex、condition RAII封装方式

```
class Mutex{
	Mutex(){
		pthread_mutex_init(NULL,&m_mutex);
	}
	void lock(){
		pthread_mutex_lock(&m_mutex);
	}
	void unlock(){
		pthread_mutex_unlock(&m_mutex);
	}
	~Mutex(){
		pthread_mutex_destory(&m_mutex);
	}
private:
	pthread_mutex_t m_mutex;
};
```
##Item 14 小心处理资源的复制
>* 常见的RAII class的copying行为是，禁止copy

##Item 15 提供一个访问原始资源的接口
>* 简单地说就是应该为资源管理类提供get()接口以访问raw resources
>* BTW，使用get,*,->，其返回的指针来访问叫做显式转换，下文这种叫隐式转换(没用过，感觉会留坑)
>* 上面说的get()接口很烦，用隐式转换代替之，上代码
```
class Font{
public:
	operator FontHanle()const{ // 隐式转换函数
		return f;
	}
private:
	FontHandle f;
};
void changeFontSize(FontHandle f,int size)//如果你用很多类似的C API
//然后调用就方便了
changeFontSize(f.get(),newSize);// 很烦的方法
changeFontSize(f,newSize);//利用隐式转换
```
##Item 16 new和delete成对使用相同的形式(form)
```
//先上bug代码
string *pstr = new string[100];
delete pstr;// bug here
```
>* 如上代码，这是未定义行为哦，至少，99的string对象没有被适当地删除，怎么这样，C++不能统一处理吗？
>* 编译器可以统一处理，但不会这么做，因为是额外的overhead(系统开销)
>* 编译器的做法是在数组的内存区域前加了[N]表明数组长度
>* 对于non-POD类型需要一个额外的长度来保存对象的个数，因此增加了开销[^ref].
[^ref]: [zhihu上陈硕先生的回答](http://www.zhihu.com/question/25438329/answer/36980855)

##Item 17 新写一行塞入share_ptr中的new obj
>* why? 因为c++的编译器没有以特定顺序完成函数参数的核算（java、C# DID）

```
processWidget(std::tr1::shared_ptr<Widget>(new Widget), priority());
```
>* 上述代码可能是先调用new--->调用priority()-->调用shared_ptr的构造函数
>* 问题是priority()可能会抛出异常，于是资源泄露
##Item 18 接口不该反直觉，让其容易使用
>* 大师觉得接口应该很容易使用对，很难用错，作为被坑过几次的人，实在感同身受i。先上奇怪的代码
```
class Date{
//容易顺序搞错的接口
		Date(int year,int month,int day);		
};
//大师说，不如。。。用对象限制输入，只是你要写更多代码
class Year{
	explicit Year(int y):y(m_y){}//显式构造only
};
//调用时
Date d(Year(2015),Month(12),Day(1));// yeah,looks better
```
>* 数据对象的行为要尽可能符合内建对象(比如int)的行为，如连续赋值什么的
>* 接口的名字和意义要尽可能一致，比如stl的各个容器都有size，但.NET平台有count，size，length......深井冰.
>* 多用shared_ptr解决问题，比如用户忘记去释放通过函数返回的new指针的函数.
##Item 19 设计class犹如设计type
>* 对象的创建和销毁方法，关系到new、new[]、delete、delete[]
>* 赋值和构造函数的不同，见item 4
>* 注意拷贝构造函数的实现 (注意nocopyable的实现)
>* 非法输入的处理，抛出异常？
>* 注意继承体系，析构函数是否应该为虚，参考[item7](#jump7)，btw，如何禁止继承?用虚拟继承+friend+private 构造函数
>* 小心对象与其他类型对象的转换，如显式和隐式转换
>* 操作符和函数是否合理，是否真的需要？后续有细说，[item23](#jump23)、[item24](#jump24)、[item26](#jump26)
>* 
##Item 20 用pass-by-reference-const
>* 除了POD，和stl容器迭代器，函数对象，都用传引用的方式吧
##Item 21 该返回value时，别返回引用
>* 有些人返回的reference是local stack上的对象....oops...
>* 那返回new在heap上的对象如何？假设有些人类相当谨慎、不会忘记delete，但如果是以下代码：
```
class Rational{
	const Rational& operator*(a,b)//简化
	{
		//比返回local更糟糕的代码
		Rational *ret = new Rational(a*b);
		return *ret;
	}
};
//大师说，当有人类这样调用时，就算记得delete，也会内存泄露
Rational a,b,c,r;
r=a*b*c;//这实际代码是
operator*(operator*(a,b),c);// oh,new了两次，而且调用者没法delete。。。。。。
```
>* 对于operator*这类函数，无论返回local-stack对象函数heap-new对象，都会因为对返回结果调用构造函数而受到惩罚
```
//一个必须返回新对象的函数的正确做法，un，那就返回一个对象呗
inline const Rational operator*(a,b)
{
	return Rational(a,b); //可能要承受构造和析构成本
}
//现代编译器其实可以安全地消除构造函数和析构函数，ROV嘛
```
##Item 22 数据成员应该声明为private
>* 额，我觉得我能自己做判断....
##<span id="jump23">Item 23</span> 多使用非友元函数非成员函数
>* 理由之一是封装度，数据成员被越少的代码访问到，该成员的封装程度就越高
>* 另一个理由是编译依赖项的减少，看代码，如果要添加新的util函数，直接在合适的地方声明，定义之，原有class不必改动，un..好处大概就是不必改别人的class。。或者有利于ABI兼容
```
// code in class_a.h
namespace  AllAboutClassA  {
   class  ClassA  { // ..};
}

// code in util_1.h
namespace  AllAboutClassA  {
   void  WrapperFunction_1() { // ..};
   //加函数。。。
}


// code in util_2.h
namespace  AllAboutClassA  {
   void  WrapperFunction_2() { // ..};
}
// ..
```
##<span id="jump24">Item 24</span>若函数所有参数都要类型转换，用非成员函数
>* 先看代码才能知道理由
```
//又是有理数的例子。。好烦
class Rational{
public:
	Rational(int n,int d= 1);//没有使用explicit，于是允许隐式转换构造
	int numerator()const;
	int denominator()const;
	//问题函数来了
	Rational operator*(const Rational& rhs)const;
};
//引发的问题举例
Rational a(1),b(2);
Rational ret = a*b;// ok，隐式转换
Rational ret = a*2;// ok，隐式转换等效于：
Rational ret = 2*a;//NO...why?
// 编译器知道你在传递int，但函数需要Rational，重要的是他知道只要给Rational构造函数赋值那个int，就能如愿以偿，so he did it.
Rational ret = a.operator*(Rational tmp(2));
//如果Rational构造函数添加explicit，a*2将错误
Rational ret = a*2;//错误，在explicit下无法将2转换为一个	Rational
```
>* 大师说，加explicit吧，至少乘法运算行为一致了，只是无法编译通过而已，但大师是不会让我们失望滴，对不对？
```
//添加一个no member function
const Rational operator*(const Rational&lhs,
						cont Rational& rhs)
{
return Rational (lhs.numerator()*rhs.numerator(),
				lhs.denominator()*rhs.denominator());
}
//应该成为友元函数吗？大师说，no,operator*根据Rational提供的接口完成了功能...
//对了大师还说，后面涉及template时，以上说法就不一定啦，item 46再见
```
##Item 25 比std::swap效率更高的swap
>* std::swap的问题是，拷贝三次，交换，某些情况下，效率太低了，上代码
```
class WidgetImp{
private:
	vector<double> v;//瞧，复制很慢的
	int a,b,c;
};
//考虑这种pimp(item31再见)的class设计场景
class Widget{
	Widget&operator=(const Widget& rhs){
		...
		*m_pImpl= *(rhs.m_pImpl);
		...
	}
private:
	WidgetImp* m_pImpl;
};
//于是，当使用std::swap时，copy了三次，其实我们只要置换m_pImpl指针就够了

//让std::swap针对swap特化....
namespace std{
	template<>//针对widget的特化版本
	void swap<Widget>(Widget&a,Widget&b)
	{
	//un..编译不过，他是私有的，但大师总有特殊技巧
		swap(a.m_pImpl,b.m_pImpl);
	}
}

class Widget{
	void swap(Widget& rhl)//用友元~
	{
		std::swap;
		swap(m_pImpl,rhl.mImpl);
	}
};
```
>* C++只允许偏特化一个class template，而不允许partially specialize一个function template..why?and what the hell
>* todo:果然不擅长模版泛型编程的topics....i will come back
##<span id="jump26">Item 26</span>变量用到再定义
>* 我也这样的，代码比较易读，先上代码
```
//一个过早定义的例子
void encryptStr(const string& pw)
{
	string encryptRet;//过早定义，浪费了构造函数的时间
	if(pw.length()<MINI_LENGTH){
		throw logic_error("password is too short");		
	}
	return encryptRet;//
}
```
>* 定义变量包含了该变量对象的构造操作，如果因为某个原因(如抛出异常，条件语句未执行等)而没有真正用到这个变量，那么构造该变量所耗费的时间和资源就白费了。
>* 在即将使用变量前再定义它对理解代码也有好处：要想知道某个变量时做什么用的？读接下来的代码便是。
>*循环里面咋办？
##<span id="jump32">Item32 </span>
##<span id="jump39">Item39 </span>

##reference
>* [blog](http://www.cppblog.com/note-of-justin/category/12550.html?Show=All)
>* [本文编辑地址](https://www.tumblr.com/edit/109397999486)

