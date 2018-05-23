---
title : STL源码学习（一）迭代器概念与traits编程技法
date : 2018-01-20 19:24
categories : STL
tags : STL
---



STL的中心思想在于：将数据容器和算法分开，彼此独立设计，最后再以一贴胶着剂将它们撮合在一起。这种胶着剂便是迭代器，迭代器设计的目的在于可以方便的获取(遍历)容器中的数据同时又不暴露容器的内部实现。

比如说，当使用一个链表迭代器时，使用者根本不会知道数据在链表中是如何存储的。这带来的好处是，所有的STL算法只需要接收迭代器最为参数就够了，根本不用管是哪个容器的迭代器，只要能够获取数据，通过迭代器改变数据就可以了。

<!--more-->

不过，迭代器的实现并不是那么简单，因为在通常使用迭代器时，可能还需要知道迭代器指向的数据类型，比如某些算法中需要定义一个同类型的变量。这就产生了矛盾，因为既不想暴露底层数据，又必须知道数据的类型，后面会看到，通过traits技法，可以很容易获取迭代器所指类型

# 类型萃取

## traits技法

作为容器专属的迭代器，容器中保存的数据类型迭代器是知道的(因为需要作为模板参数)，所以如果在迭代器中定义内嵌类型，就可以获取迭代器所指数据的类型，比如定义MyIter迭代器

```c
template <class T>
struct MyIter
{
	/* 定义T的别名 */
  	typedef T value_type;
  	T* ptr;
  	MyIter(T* p = 0) : ptr(p) {}
  	T& operator*() const { return *ptr; }
  	//...
};

templace <class I>
/* 使用I的内嵌了类型获取迭代器所指数据类型 */
typename I::value_type
func(I ite)
{
  	return *ite;
}

//...
MyIter<int> ite(new int(8));
cout << func(ite);	//输出8
```

>当使用作用域符号表达一个类型时，需要在前面添加typename显式指出这是一个型别。因为I是一个模板参数，在具现化之前，编译器对I一无所知，根本不知道I::value_type是什么，所以需要显式告诉编译器，这是一个型别



Traits就是对上述内嵌类型获取迭代器所指数据类型方法的封装，形象地称为"萃取"迭代器特性，而value_type正是迭代器特性之一

```c
/* 用于萃取迭代器类型，迭代器指向元素的类型等
 * 使用时可以直接使用
 * iterator_traits<I>::value_type获取迭代器指向元素类型 */
template <class Iterator>
struct iterator_traits {
  typedef typename Iterator::value_type        value_type;
};
```

通过这个萃取器，如果Iterator声明有value_type内嵌类型，那么就可以使用如下方法获取迭代器Iterator所指类型

```c
templace <class I>
/* 函数返回型别，通过iterator_traits从I中萃取出它保存的数据类型 */
typename iterator_traits<I>::value_type	
func(I iter)
{
  	return *iter;
}
```

## 指针特化版本

多了一层间接性带来的好处是可以为iterator_traits定义特化版本，由于原生指针也可以算作迭代器的一种，所以像iterator_traits\<int *\>::value_type这样的式子也应该是合理的，但是由于原生指针没有内嵌类型，会导致上述式子不能通过编译。解决方法是为原生指针设计特化版本，使上述式子合法

```c
/* 针对原生指针的偏特化版本，因为指针没有内嵌类型 */
template <class T>
struct iterator_traits<T*> {
  typedef T                          value_type;
};

/* 针对常量指针的偏特化版本 */
template <class T>
struct iterator_traits<const T*> {
  typedef T                          value_type;
};
```

有了上面的特化版本，当使用iterator_traits\<int *\>::value时，就可以萃取出int型别

>为了能让"特性萃取机"traits正常工作，规定每个迭代器都必须以内嵌型别定义的方式定义出相应型别(比如value_type)

# 内嵌类型

## 定义

迭代器部分只有iterator_traits萃取功能不容易理解，剩下的部分是关于迭代器型别的，包括

* value type，迭代器指向的数据值类型
* different type，迭代器差类型
* reference type，迭代器指向的数据引用类型
* pointer type，迭代器指向的数据指针类型
* iterator_category，迭代器类型

这几种类型在萃取器iterator_traits中也都有相应的定义，用于萃取出相应型别，同上述的valut_type一样

```c
/* 用于萃取迭代器类型，迭代器指向元素的类型等 */
/* 使用时可以直接使用
 * iterator_traits<I>::iteartor_category获取迭代器I的类型
 * iterator_traits<I>::value_type获取迭代器指向元素类型
 * 依次类推 */
template <class Iterator>
struct iterator_traits {
  typedef typename Iterator::iterator_category iterator_category;
  typedef typename Iterator::value_type        value_type;
  typedef typename Iterator::difference_type   difference_type;
  typedef typename Iterator::pointer           pointer;
  typedef typename Iterator::reference         reference;
};
```

* value type实际上就是迭代器所指对象的型别，也就是模板参数T。假设使用MyIter<int> it迭代器，那么value type就是int


* difference type是指迭代器之间的距离类型，如两个迭代器作差，保存迭代器之间的距离等，其类型就是difference type，这个类型通常是ptrdiff_t类型


* reference type是指迭代器所指对象的引用类型，通常是T&


* pointer type是指迭代器所指对象的指针类型，通常是T*

```c
  typedef T                          value_type;
  typedef ptrdiff_t                  difference_type;
  typedef T&                         reference;
  typedef T*                         pointer;
```

## 迭代器类型

在STL中，根据迭代器所支持的算术运算，实际上存在多种类型的迭代器，分别是

* input iterator只读迭代器，不可以改变迭代器所指对象。只支持++操作
* ouput iterator只写迭代器，允许改变迭代器所指对象。只支持++操作
* forward iterator前向迭代器，可读写。只支持++操作
* bidirectional iterator双向迭代器，可读写。支持++和--操作
* random access iterator随机迭代器，可读写。支持所有指针算术操作，包括+,-,+=,-=,++,--等

可以看到，不同类型的迭代器可以提供不同的功能，但这同时也带来了一些问题，以advance函数为例，该函数将给定的迭代器移动n步，然而，不同类型的迭代器支持不同的算术操作，如果都使用++，那么对随机迭代器而言就过于耗时(因为本可以直接使用+=)。这就是iterator_category迭代器类型的作用，结合模板的参数推导区分不同的迭代器类型，同时采用最有效的方法进行算术操作

模板函数参数推导功能保证根据参数类型选择最合适的重载函数，所以在STL中，提供了五个类定义分别对应五个迭代器类型

```c
/* 定义迭代器类型，每个迭代器中都会将自己迭代器类型typedef成iterator_categories
 * 由于C++不支持直接获取类型操作，所以需要利用模板参数推导实现不同类型的不同操作 */
struct input_iterator_tag {};
struct output_iterator_tag {};
struct forward_iterator_tag : public input_iterator_tag {};
struct bidirectional_iterator_tag : public forward_iterator_tag {};
struct random_access_iterator_tag : public bidirectional_iterator_tag {};
```

这五个类定义的继承关系和迭代器之间的继承关系相同，另外这五个类是空类，因为它的作用仅仅是作为模板参数用于函数重载

在迭代器的定义中会根据自身所支持的功能最强的迭代器类型定义iterator_category，如

```c
  typedef random_access_iterator_tag iterator_category;
```

这样，当再次使用advance函数时，传入自己的迭代器类型对应的类，同时根据模板参数推导就可以调用不同的函数

```c
/* 将迭代器i移动n步，n可正可负 */
template <class InputIterator, class Distance>
inline void advance(InputIterator& i, Distance n) {
    /* 不同迭代器支持的操作不同
     * 只读，只写，正向迭代器只支持++操作
     * 双向迭代器支持++，--操作
     * 随机迭代器支持+=，-=，++，--操作
     * 为了根据不同迭代器选择不同的__advance函数，采用模板参数推导的方法实现重载 */
    /* iterator_category()也是一个模板重载，用于创建一个迭代器i的tag对象
     * 根据模板参数推导，可以实现__advance重载 */
  __advance(i, n, iterator_category(i));
}
```

> 模板参数中迭代器类型是InputIterator类型的原因是STL将模板参数类型设置为最弱的类型(类似继承体系中的基类)，实际类型会根据i实际的类型判断(类似于继承体系中的多态)

iterator_category函数用于获取迭代器对应的tag对象

```c
/* 返回迭代器类型对象，注意返回的是一个类对象
 * 这是因为模板参数推导只针对类对象进行推导*/
template <class Iterator>
inline typename iterator_traits<Iterator>::iterator_category
iterator_category(const Iterator&) {
  typedef typename iterator_traits<Iterator>::iterator_category category;
  return category();
}
```

至此，根据__advance(i, n, iterator_category(i))函数调用的第三个参数类型，模板参数推导可以实现调用不同的函数

```c
/* 针对只读迭代器的__advance重载 */
template <class InputIterator, class Distance>
inline void __advance(InputIterator& i, Distance n, input_iterator_tag) {
  while (n--) ++i;
}


/* 针对双向迭代器的__advance重载 */
template <class BidirectionalIterator, class Distance>
inline void __advance(BidirectionalIterator& i, Distance n, 
                      bidirectional_iterator_tag) {
  if (n >= 0)
    while (n--) ++i;
  else
    while (n++) --i;
}

/* 针对随机迭代器的__advance重载 */
template <class RandomAccessIterator, class Distance>
inline void __advance(RandomAccessIterator& i, Distance n, 
                      random_access_iterator_tag) {
  i += n;
}
```

>STL中只提供了三个\_\_advance函数重载，这是因为上述的继承体系。由于output迭代器，forward迭代器的\_\_advane操作和input迭代器的操作相同，所以完全可以向上调用input迭代器的对应函数(好比调用派生类的某个函数，但是派生类中不存在这个函数，就会向上调用基类的对应函数)

# 小结

迭代器部分的难点有两个，一个是通过traits技法实现类型萃取，另一个是通过模板参数推导实现函数重载。不过真正看过源码后会发现其实并不困难，当然，STL需要内部的迭代器都遵循它的规定，即定义上述的五个型别