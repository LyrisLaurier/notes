> 基础直接跳过，从P84开始看，也就是面向对象部分：[01 程序的内存模型-内存四区-代码区._哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1et411b73Z?p=84&vd_source=8e66d0be2e227fad85c74192f79799d6) 

# 核心编程

## new/delete

内存分区模型

* 代码区：存放函数体的二进制代码，由OS管理
* 全局区：存放全局变量、静态变量、常量。在程序结束后由OS释放
* 栈区：编译器自动分配释放，存放函数的参数值、局部变量等
* 堆区：程序员分配释放，程序结束OS也会自动回收

堆区数据：new → 堆区

```c++
int main() {
    int* a = new int(10); // new关键字创建的数据在堆区
    cout << a << ":" << *a << endl; // 0x197997018e0:10
    return 0;
}
```

new返回的是该数据类型的一个指针

```c++
int* b = new int[10]; // new数组
for (int i = 0; i < 10; ++i) {
    b[i] = i + 100;
    cout << "b[" << i << "]:" <<b[i] << endl;
}
```

释放用 delete，delete后再读取该块内存会引发异常。

```
delet a;
delete[] b;
```

## 引用

格式

```c++
int &b = a; //int* const b = &a;
```

注意

* <u>引用必须要初始化，且初始化后不可再更改。</u>`b=c` 是赋值，不是更改引用
* 在函数调用中用引用，可以改变原参数的值。引用本质就是指针
* 不能返回局部变量的引用，很可能返回不出来正确值（如果能打印出来可能是因为编译器自动保留,这种情况一般第二次就打不出来了。也可能直接就没内容）

```c++
void change1(int &a){ //要是想防止修改就用 const int &a
    a=20; //可以改变a的值
}
int main() {
    int a = 0;
    change(a);
    cout << a << endl; //a=20
    return 0;
}
```

常量引用需要加 const关键字

```c++
const int& ref = 10; // int temp = 10; const int& ref = temp;
```

## 函数

默认值：函数缺省就用默认值，传了参数就用传的值。规定默认值后的所有参数都要给默认值。<u>函数声明和实现中只能有一个设置默认参数。</u>

占位参数：`int fun2(int a,int){...}` 

函数重载：重载条件是函数名相同，但参数个数/顺序/类型不同。返回值不同不能作为重载条件

## 类和对象

### 封装

<u>封装、继承、多态</u>

一个最简单的类以及调用

```c++
class Circle{
public://访问权限
    //属性
    float r;
    //行为
    float getCircle(){
        return 2 * 3.14 * r;
    }
};

int main(){
    cout << "sample.cpp" << endl;
    Circle circle;
    circle.r = 10.0;
    cout << "circumference: " << circle.getCircle() << endl;
    return 0;
}
```

其中有三种访问权限

* public-公有
* protected-保护：类内成员可以访问，类外不行；继承中子可以访问父
* private-私有：类内可以类外不行；继承中子不可以访问父

struct＆class

* struct 默认权限 public
* class 默认权限 private

一般将成员属性设置为私有：将属性设置为私有，另外设置一些 public 方法进行读/写操作

### 构造函数&析构函数

#### 构造函数

构造函数： `类名(){}`

* 可以有参数，因此可以重载
* 没有返回值也不写void
* 调用对象时自动调用构造，不用手动调用，且只会调用一次。即初始化时就调用

析构函数： `~类名(){}`

* 没返回值也不写void
* 不可以有参数，也不可以重载
* 对象销毁前自动调用析构，不用手动调用，且只会调用一次

调用规则：编译器默认提供三个函数。自定义有参构造函数后，不再提供默认无参构造；自定义拷贝构造后，不再提供默认的其他构造函数

* 默认构造：空实现
* 析构函数：空实现
* 拷贝构造：值拷贝

```C++
class Person{
public:
    Person(){
        cout << "Person()" << endl;
    }
    ~Person(){
        cout << "~Person()" << endl;
    }
};

int main() {
    Person p; //Person()和~Person()都输出
    return 0;
}
```

> 都调用的原因是，p是栈上数据，自动释放后会调用析构

分类

* 有参/无参构造
* 普通/拷贝构造

构造函数

```c++
class Person{
public:
    int age;
    //无参构造
    Person(){ 
        cout << "Person()" << endl;
    }
    //有参构造
    Person(int a){ 
        cout << a << endl;
    }
    //拷贝构造
    Person(const Person &p){ 
        age = p.age;
    }
};

int main() {
    Person p;
    p.age = 10;
    Person p1(p); //Person p1 = Person(p)
    cout << "p1.age:" << p1.age << endl;
    return 0;
}
```

构造函数调用

* 括号：

  ```C++
  Person p; //Person p()不行,不能创建对象,会被认为是一个函数声明
  Person p1(10);
  //Person p2(p1)不行,会等价于Person p2
  ```

  <u>针对调用默认构造时不能加括号；不要用拷贝构造函数初始化匿名对象</u> 

* 显示：

  ```C++
  Person p;
  Person p1 = Person(10); //有参构造
  Person p2 = Person(p2); //拷贝构造
  ```

* 隐式转换：

  ```C++
  Person p4 = 10; //Person p4 = Person(10)
  Person p5 = p4;
  ```

匿名对象：当前行结束就销毁（执行析构）。

#### 初始化列表

传统的初始化，声明变量后通过构造函数传入参数、赋值可以进行初始化

```C++
class Test{
public:
	int m_a;
	int m_b;
	int m_c;

	//传统方式
    Test(int a, int b, int c){
		m_a = a;
		m_b = b;
		m_c = c;
	}

	//初始化列表
    Test(int a, int b, int c):m_a(a),m_b(b),m_c(c){
	}
};
```

#### 拷贝构造函数

拷贝构造函数调用

* 使用一个已经创建完的对象来初始化一个新对象

  ```C++
  Person p1(10);
  Person p2(p1);
  ```

* 值传递的方式给函数参数传值

  ```C++
  void test_person(Person p){
  	cout << "test_person()" << endl;
  }
  int main()
  {
  	Person p;
  	test_person(p); //值传递,这里拷贝了
  	return 0;
  }
  ```

* ~~以值的方式返回局部对象~~：已经被优化了

  ```C++
  Person test2(){
  	Person p1(5);
  	cout << (int*)&p1 << endl;
  	return p1;
  }
  
  int main()
  {
  	Person p = test2();
  	cout << (int*)&p << endl;
  	return 0;
  }
  ```

  > ~~这里存疑，视频里说是值传递，本地运行出来是整个对象传递~~
  >
  > 已解决，[C++ 函数返回对象时并没有调用拷贝构造函数_shang_ch的博客-CSDN博客](https://blog.csdn.net/nbu_dahe/article/details/119142610) 中有解释：其原因是 RVO（return value optimization），被G++进行值返回的优化了，可以对g++增加选项 `-fno-elide-constructors` 将RVO优化关闭

#### 深拷贝&浅拷贝

浅拷贝：简单的赋值拷贝；编译器提供的拷贝构造函数就是钱拷贝

深拷贝：在堆区重新申请空间进行拷贝操作

浅拷贝：

```C++
class Person
{
public:
	int age=18;
	int* n;

	Person(int a, int number)
	{
		cout << "Person(int a)" << endl;
		age = a;
		n = new int(number); //开到堆区
	}
};

int main()
{
	Person p1(19,2);
	Person p2(p1);
	p1.age = 20;
	*p1.n = 3;
	cout << "p1.age:" << p1.age << "; p1.n:" << *p1.n << endl; //p1.age:20; p1.n:3
	cout << "p2.age:" << p2.age << "; p2.n:" << *p2.n << endl; //p2.age:19; p2.n:3
	return 0;
}
```

* 这里面 age、n 属性都进行浅拷贝，p2.n 会随着 p1.n 改变，查地址可以看到这两个 n 是一个地址

> 有创建堆区，就要记得用析构释放堆区，但是浅拷贝释放堆区空间时会出现问题

```C++
~Person()
{
    if (n != NULL){
        delete n;
        n = NULL; //防止野指针出现,做置空
    }
    cout << "~Person" << endl;
}
```

* p2 释放一次 n 的堆区，p1 再释放一次就重复了 → 深拷贝解决

深拷贝：这样 p1 的修改就不会影响 p2 的值（指针保存的内容）

```C++
Person(const Person &p)
{
    cout << "Person(const Person &p)" << endl;
    age = p.age; //浅拷贝
    n = new int (*p.n); //深拷贝
}
int main()
{
	Person p1(19,2);
	Person p2(p1);
	p1.age = 20;
	*p1.n = 3;
	cout << "p1.age:" << p1.age << "; p1.n:" << *p1.n << endl; //p1.age:20; p1.n:3      
	cout << "p2.age:" << p2.age << "; p2.n:" << *p2.n << endl; //p2.age:19; p2.n:2 
	return 0;
}
```

也就是，<u>如果有开辟堆区数据，就要自己写拷贝函数实现深拷贝，防止出问题</u>

#### 对象成员

C++ 中允许类的成员是另一个类对象，即对象成员

```C++
class A {}
class B {
    A a;
}
```

```C++
class Test{
public:
	int m_a;
	Person m_p;
	Test(int a, Person p): m_a(a), m_p(p) {
	}
};

int main()
{
	Person p(18,1);
	Test t(5,p);
	cout << "m_a:" << t.m_a << endl;
	cout << "m_p.age:" << t.m_p.age << endl;
	cout << "m_p.n:" << *t.m_p.n << endl;
	return 0;
}
```

* 构造函数中会先创建 Person 再创建 Test，析构函数顺序相反

#### static

静态成员变量

* 所有对象共享一份数据。可以更改，只是一改就全改了

  ```C++
  Person p1;
  Person p2;
  p2.a = 20;
  cout << p1.a << endl; //p1.a = p2.a都被改了
  ```

* 编译阶段分配内存

* 类内声明，类外初始化

  ```C++
  class Person{
  public:
  	static int a;
  };
  int Person::a = 10; //int a就成全局变量了,用::限制作用域
  ```

静态成员变量还可以通过类名访问： `cout << Person::a << endl;` ，当然如果是 private 就访问不到了

静态成员函数

* 所有对象共享一个函数
* 静态成员函数只能访问静态成员变量

静态成员函数也可以通过对象、类名访问

```C++
class Person{
public:
	static void func(){
	cout << "static void func()" << endl;
	}
};

int main()
{
	Person p1;
	p1.func();
	Person::func();
	return 0;
}
```

