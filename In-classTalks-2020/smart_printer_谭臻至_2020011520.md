# 智能指针

## 内存的分配

计算机程序中内存通常被分为*静态内存*、*堆*和*栈*等多种不同形式的内存。通常静态内存和栈是由系统自动分配的，而堆上的内存可以由程序控制调用。

通常当堆中的内存不再使用时，我们需要对其进行销毁（释放），以避免在程序运行的过程中发生内存溢出。如果内存溢出过多可能影响程序的正常运行，甚至导致程序崩溃

在C++中，动态分配的内存是通过`new`和`delete`关键字来实现的

但是我们在编写程序的时候往往会忘记delete申请的内存，或者在一块内存已经被释放的情况下重复释放了这块内存。前者会导致程序出现内存溢出的问题，而后者则会直接导致程序崩溃。

于是我就想，如果我每写一个`new`就写一个`delete`，完全不犯错，那么是否就可以杜绝由于内存分配失误导致的程序错误了呢？

事实上，这个功能已经在系统库中得到了实现，这就是在C++11以后引入的**智能指针**

## 智能指针

智能指针本身是一个对象，但是通过运算符重载等方式，可以保留一般指针的使用方式。

智能指针相对于普通指针，最大的作用就是增加了可以自动释放对象的特性，尽可能的保证了内存的安全性

智能指针被定义在`memory`头文件中，目前最常用的三种智能指针有`shared_printer`、`unique_printer`、`weak_printer`

```C++
shared_ptr<typename _Tp>
unique_ptr<typename _Tp>
weak_ptr<typename _Tp>

```

### `shared_ptr`

智能指针中使用最广泛的是shared_ptr，它可以让多个对象指向同一块内存区域。

#### 工作原理

shared_ptr实现自动析构的原理是在类中利用一个隐藏的成员变量进行计数，我们称之为引用计数

当shared_ptr进行一次拷贝，则所有指向该块内存的智能指针计数器都会递增

而当shared_ptr进行一次赋值或者析构，则该指针的计数器归零，同时其他指向该块内存的智能指针计数器都会递减

当一个shared_ptr的计数器变为0时，就会自动调用析构函数销毁这些对象

智能指针本质上是一个对象，但是通过一系列封装，使得其表现得和普通的指针类似。这其实是利用了RAII的思想

#### 成员函数

| 成员函数                | 作用                                                      |
| ----------------------- | --------------------------------------------------------- |
| `p`                     | 可用作指针是否为空的判断，若为空则返回false，否则返回true |
| `*p`                    | 解引用p，获得p指向的对象                                  |
| `p.get()`               | 获得p的地址                                               |
| `p.swap()`              | 将p和q的地址进行交换                                      |
| `make_shared<T> (args)` | 动态分配一个shared_ptr对象，并使用args初始化              |
| `p.use_count()`         | 返回与p共享指针的智能指针数量（可能很慢，主要用于调试）   |
| `p.unique()`            | 若p.use_count()=1返回true，否则返回false                  |

`shared_ptr`有一系列成员函数，可以执行不同的功能。

前两项和普通指针相同，`get`函数可以获得`p`的地址，`swap`函数可以进行地址的交换（实质上是利用引用进行交换）

`make_shared()`函数比较重要，通常我们应该使用这个函数对`shared_ptr`进行初始化。

`use_count()`函数返回与p共享的智能指针数量，即引用计数的值，但是由于其实现较为复杂，会大幅降低程序的运行效率，因此通常只是在调试中使用

#### 初始化

如上所述，可以使用`make_shared`对智能指针进行初始化。以下是几种初始化的方式

如果在括号中不填写内容，编译器则会做默认初始化

注意：`shared_ptr`是一个对象，因此不可以使用某一种类型变量的指针对其进行初始化

```C++
shared_ptr<int> number1 = make_shared<int>(42);
shared_ptr<int> number2 = make_shared<int>();
shared_ptr<string> sentence = make_shared<string>("hello!");

```

#### 计数

```C++
int main(){
	int val_1 = 10;
	shared_ptr<int> ptr_1 = make_shared<int> (val_1);
  shared_ptr<int> ptr_change (ptr_1); //利用拷贝构造函数
  cout << "ptr1 use count: " << ptr_1.use_count() << endl;
   int val_2 = 20;
  shared_ptr<int> ptr_2 = make_shared<int> (val_2);
  ptr_change = ptr_2;//利用=重载拷贝构造函数
   cout << "------------------------" << endl;
  cout << "ptr1 use count: " << ptr_1.use_count() << endl;
  cout << "ptr2 use coutn: " << ptr_2.use_count() << endl;
  return 0;
}

```

如上面程序所示，我们可以看到`share_ptr`的计数情况

当生成`ptr1`之后，`ptr1`的计数为1

利用`ptr1`对`ptr_change`做拷贝构造，使得两者的计数都变为2

然后利用`ptr2`对`ptr_change`进行赋值，此时ptr1的引用计数-1，`ptr2`的引用计数+1

#### 注意事项

使用`shared_ptr`时有一些注意事项：

首先不能使用一个原始指针对多个`shared_ptr`进行初始化或者进行赋值，否则会导致同一块内存被重复释放

应该使用拷贝构造实现`shared_ptr`的复制

另外，不能用`shared_ptr`进行循环引用，即p1指向p2而p2由指向p1，这会导致内存无法被正确释放。

应该使用后面介绍的`weak_ptr`进行循环引用。

### `unique_ptr`

#### 特点

`unique_ptr`正如其名字所示的那样，一片内存区域只能有一个`unique_ptr`指向该位置（但是可以有多个`shared_ptr`和一个`unique_ptr`指向该位置）

和`shared_ptr`一样，该对象在离开作用域后会自己销毁，或者也可以利用`release()`函数进行销毁

#### 实例化和改变对象

```C++
int main(){
  unique_ptr<string> s (new string("word"));
  //unique_ptr<string> s_copy = s; //禁止赋值
  //unique_ptr<string> s_copy(s);  //禁止拷贝
  unique_ptr<string> s_move = move(s);
  if(s){
    cout << *s << endl;
  }else{
    cout << "s has been deleted." << endl;
  }
  cout << *s_move << endl;
  s_move.reset(new string("world"));
  cout << *s_move << endl;
  return 0;
}

```



在c++-14及以后的c++版本中支持使用`make_unique`函数进行实例化

如果不支持`make_unique`函数的c++版本，可以使用指针进行初始化

 注意此处和`shared_ptr`不同，因为`unique_ptr`是绑定在内存区域上的，因此可以直接使用内存区域进行初始化

可以看到，`unique_ptr`禁用了赋值和拷贝功能，仅支持通过移动构造的方式实现实例化，这保证了`unique_ptr`指向对象的唯一性

同时，我们也可以使用`reset()`函数对`unique_ptr`所指向对象进行更改

### `weak_ptr`

#### 特点

此外还有一类比较特殊的智能指针——`weak_ptr`

`weak_ptr`和前面两种指针不同，它并没有真正绑定到一块内存区域上，因此不能像通常的指针一样获取内存信息

`weak_ptr`更像是一个`shared_ptr`的观测者，它可以获取绑定对象的引用计数。

但同时，在设计`weak_ptr`时也预留了一个`lock()`函数接口，实现对观测对象地址信息的获取。

`weak_ptr`可以解决我们之前提到的循环引用的问题

#### 循环引用

```C++
class Bird{
	string bird_name;
	shared_ptr<Bird> o_bird;
public:
	Bird(const string from): bird_name(from){
		cout << "constructed '" << bird_name << "'" << endl;
	}
	~Bird(){
		cout << "destructed '" << bird_name << "'" << endl;
	}
	friend void relation(shared_ptr<Bird> &bird_1, shared_ptr<Bird> &bird_2){
		bird_1->o_bird = bird_2;
		bird_2->o_bird = bird_1;
		cout << bird_1->bird_name << " is with " << bird_2->bird_name << endl;
	}
};

int main(){
  shared_ptr<Bird> p1 = make_shared<Bird> ("Pigeon");
  shared_ptr<Bird> p2 = make_shared<Bird> ("Swift");
  relation(p1, p2);
	return 0;
}


```

在这段程序中定义了一个`Bird`类，然后利用`relation`函数将`bird_1`中的智能指针绑定到`brid_2`上，而将`bird_2`的智能指针绑定到`brid_1`上

这样可以定义一个循环引用

但是当我们运行程序时会发现，在程序退出时没有输出“destructed xxx”信息，说明对象没有被析构

造成这个结果的原因是当程序试图调用析构函数析构p1时，其绑定的智能指针p2的计数并没有归零，因此不可以析构p2。

同理，试图析构p1时也会遇到一样的情况

因此我们可以使用“旁观者”`weak_ptr`实现正确的析构

因为`weak_ptr`不可以改变该块内存，因此`shared_ptr`不会计数

同时，我们也可以使用`lock()`函数读取信息，实现其原本`shared_ptr`可以实现的内容

```C++
class Bird{
	string bird_name;
	weak_ptr<Bird> o_bird;
public:
	Bird(const string from): bird_name(from){
		cout << "constructed '" << bird_name << "'" << endl;
	}
	~Bird(){
		cout << "destructed '" << bird_name << "'" << endl;
	}
	friend void relation(shared_ptr<Bird> &bird_1, shared_ptr<Bird> &bird_2){
		bird_1->o_bird = bird_2;
		bird_2->o_bird = bird_1;
		cout << bird_1->bird_name << " is with " << bird_2->bird_name << endl;
	}
};

int main(){
  shared_ptr<Bird> p1 = make_shared<Bird> ("Pigeon");
  shared_ptr<Bird> p2 = make_shared<Bird> ("Swift");
  relation(p1, p2);
	return 0;
}

```

### C++标准中的规定

由于智能指针是在c++11中才首次引入c++标准中的，因此有必要介绍一下其在不同c++标准中的规定

之所以智能指针是在c++11中才首次出现，是因为在c++11中才首次出现了右值引用，实现了移动构造的功能。通过这个功能才能实现`unique_ptr`

而在c++11以前，c++标准中的智能指针使用的是`auto_ptr`，但是由于其存在一系列问题，如可能在变量传值时资源被转移，导致重复销毁对象；`auto_ptr`对于动态分配的数组不适用，不兼容STL等，因此`auto_ptr`在C++11中已经被弃用，且在C++17中被移除标准库

在C++-11标准中，`uniqur_ptr`可以使用类似于普通指针的方式用下标调用数组，但是`shared_ptr`不可以

可以通过使用`get()`函数的方式获取指针地址，然后使用数组操作，但是这样操作可能导致指针悬空

另外就是C++-11标准中并没有对应于`unique_ptr`的生成函数，这一缺陷在C++-14中得到了修改

