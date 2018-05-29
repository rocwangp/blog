---
title: C++代码片段（三）过滤可变模板参数中某些类型的变量
date: 2018-05-29
categories: C++
tags: C++
---



将可变模板参数列表中的某些类型过滤掉，然后返回剩下数据的元组。用到了上一篇中的[判断可变模板参数中是否包含某一特定类型](https://rocwangp.github.io/2018/05/26/C++%E4%BB%A3%E7%A0%81%E7%89%87%E6%AE%B5%EF%BC%88%E4%BA%8C%EF%BC%89%E5%88%A4%E6%96%AD%E5%8F%AF%E5%8F%98%E6%A8%A1%E6%9D%BF%E5%8F%82%E6%95%B0%E5%88%97%E4%B8%AD%E6%98%AF%E5%90%A6%E5%8C%85%E5%90%AB%E6%9F%90%E4%B8%80%E7%89%B9%E5%AE%9A%E7%B1%BB%E5%9E%8B/)的方法

<!--more-->

```c++
#include <iostream>
#include <tuple>
#include <string>
#include <vector>
#include <list>

template <typename T, typename... Args>
struct contains : public std::false_type {};

template <typename T, typename U, typename... Args>
struct contains<T, U, Args...> :
    public std::conditional_t<std::is_same_v<T, U>, std::true_type, contains<T, Args...>> {};

template <typename... FilterArgs>
struct type_filter
{
    static constexpr auto filter() { return std::tuple{}; }

    template <typename T, typename... Args>
    static constexpr auto filter(T&& t, Args&&... args) {
        using type = std::remove_reference_t<T>;
        if constexpr (contains<type, FilterArgs...>::value) {
            return filter(std::forward<Args>(args)...);
        }
        else {
            return std::tuple_cat(std::make_tuple(std::forward<T>(t)), filter(std::forward<Args>(args)...));
        }
    }
};
```



## 实现原理

相当于依次遍历可变模板参数列表中的数据，每次遍历时判断数据的类型是否是想要过滤掉的类型（也就是判断当前类型是否存在于过滤类型中），如果是则丢弃当前数据继续遍历，否则保留数据

没有过滤掉的数据保存在元组中，采用std::tuple_cat方法将两个std::tuple连接成一个

```c++
std::tuple<int, double> t1{ 10, 2.5 };
std::tuple<std::string> t2{ "abc" };
auto t = std::tuple_cat(t1, t2); // decltype(t) = std::tuple<int, double, std::string>
```



## 需要注意的问题

过滤函数filter的声明为

```c++
template <typename T, typename... Args>
static constexpr auto filter(T&& t, Args&&... args);
```

接收的参数类型是T&&，所以分成两种情况讨论（假设传入的类型是int）

* 传入的t是左值引用类型int&，则T会被推导成int&，由于引用叠加效果，int&&& => int&
* 传入的t是右值引用类型int&&，则T会被推导成int，不存在引用叠加效果，int&& => int&&

而在调用contains<T, Args...>判断T是否出现在Args...中时，如果T是引用类型，就无法正确比较，也就是下面的这种情况

```c++
if constexpr (std::is_same_v<int&, int>) {
    std::cout << "int& is same as int\n";
}
else {
    std::cout << "int& is different from int\n";
}

// 输出：int& is different from int
```

所以需要在调用contains之前去掉类型T上面的引用属性，使用标准库实现

```c++
std::remove_reference_t<T>
```



## 测试程序

```c++
#include <iostream>
#include <tuple>
#include <string>
#include <vector>
#include <list>

template <typename T, typename... Args>
struct contains : public std::false_type {};

template <typename T, typename U, typename... Args>
struct contains<T, U, Args...> :
    public std::conditional_t<std::is_same_v<T, U>, std::true_type, contains<T, Args...>> {};

template <typename... FilterArgs>
struct type_filter
{
    static constexpr auto filter() { return std::tuple{}; }

    template <typename T, typename... Args>
    static constexpr auto filter(T&& t, Args&&... args) {
        using type = std::remove_reference_t<T>;
        if constexpr (contains<type, FilterArgs...>::value) {
            return filter(std::forward<Args>(args)...);
        }
        else {
            return std::tuple_cat(std::make_tuple(std::forward<T>(t)), filter(std::forward<Args>(args)...));
        }
    }
};

template <typename Tuple, typename Func, std::size_t... Idx>
void for_each(Tuple&& t, Func&& f, std::index_sequence<Idx...>) {
    (f(std::get<Idx>(t)), ...);
}

template <typename... Args>
void filter_test(Args&&... args) {
    auto t = type_filter<int, std::list<std::string>, std::vector<int>>::filter(std::forward<Args>(args)...);
    for_each(t, [](auto& item) {
        std::cout << item << std::endl;
    }, std::make_index_sequence<std::tuple_size_v<decltype(t)>>{});
}

int main()
{
	// l被推导为左值引用std::list<std::string>&
	// n被推导为左值引用int&
	// 字面值常量"abc", 20, 2.5被推导为右值引用std::string&&, int&&, double&&
	// vector被推导为右值引用std::vector<int>&&
    std::list<std::string> l{ "abc", "def", "123" };
    int n = 10;
    filter_test(l, "abc", n, 20, 2.5, std::vector<int>{1, 2, 3});
    return 0;
}
```

