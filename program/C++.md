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

### 对象的初始化和清理

构造函数： `类名(){}`

* 可以有参数，因此可以重载
* 没有返回值也不写void
* 调用对象时自动调用构造，不用手动调用，且只会调用一次。即初始化时就调用

析构函数： `~类名(){}`

* 没返回值也不写void
* 不可以有参数，也不可以重载
* 对象销毁前自动调用析构，不用手动调用，且只会调用一次

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

有参/无参构造、普通/拷贝构造



