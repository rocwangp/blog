---
title: C++代码片段（二）判断可变模板参数中是否包含某一特定类型
date: 2018-05-26 17:25
categories: C++
tags: C++
---



首先定义基础模板类，表示不包含给定类型

```c++
template <typename T, typename... Args>
struct contains : public std::false_type {};
```

接着进行偏特化，将可变模板参数中的类型逐个和目标类型进行比较，直到类型相同或者模板参数列表为空

```c++
template <typename T, typename U, typename... Args>
struct contains<T, U, Args...> :
    public std::conditional_t<std::is_same_v<T, U>, std::true_type, contains<T, Args...>> {};
```

> 由于基础模板类的模板参数仅有T和Args两个，所以如果想要表示三个就需要进行特化
>

<!--more-->

std::conditional是一个编译期类型选择模板库，使用方法如下

```c++
std::conditional<bool, type1, type2>::type
```

当布尔表达式为真时返回type1，为假时返回type2。注意布尔表达式必须可以在编译期计算出来

在contains的实现中，判断类型T和U是否相同，如果相同返回std::true_type类型，否则丢弃U，继续展开Args直到展开为空时，调用基础模板类，相当于返回std::false_type类型



测试程序

```c++
#include <tuple>
#include <vector>
#include <iostream>

template <typename T, typename... Args>
struct contains : public std::false_type {};

template <typename T, typename U, typename... Args>
struct contains<T, U, Args...> :
    public std::conditional_t<std::is_same_v<T, U>, std::true_type, contains<T, Args...>> {};

template <typename... Args>
void contains_test(Args&&... ) {
    if constexpr (contains<std::string, Args...>::value) {
        std::cout << "contains std::string type" << std::endl;
    }
    else {
        std::cout << "don't contains std::string type" << std::endl;
    }
}
int main()
{
    contains_test(10, 2.5, std::vector<int>{1, 2, 3}, std::string("hello world"), 1);
    return 0;
}
```

