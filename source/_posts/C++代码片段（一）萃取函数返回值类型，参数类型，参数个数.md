---
title: C++代码片段（一）萃取函数返回值类型，参数类型，参数个数
date: 2018-05-26
categories: C++
tags: C++
---



函数的类型主要集中在以下几种

* 函数指针
* 函数对象，是一个类对象，内部重载的operator()函数是一个函数指针
* lambda，匿名函数对象，同函数对象
* function对象

<!--more-->

> 后三者都是类对象，可以看成一种类型
>

## 定义基础模板类

```c++
template <typename T>
struct function_traits;
```



## 针对函数指针进行偏特化



对于函数指针，存在两种情况

* 直接通过decltype获取类型
* 利用模板推导类型


```c++
int pointer_func(int a, int b) {
    return a + b;
}
// decltype(pointer_func) is int(int, int)
```

```c++
int pointer_func(int a, int b) {
    return a + b;
}
template <typename Func>
void traits_test1(Func&&) {
    
}
template <typename Func>
void traits_test2(Func) {

}
int main() {
	// Func = int(&)(int, int)
    traits_test1(pointer_func);
    
    // Func = int(*)(int, int)
    traits_test2(pointer_func);
}
```



不难发现，函数指针的类型为

* R(*)(Args...)
* R(&)(Args...)
* R(Args...)

其中R是函数返回值类型，Args是函数的参数列表，针对这三种进行特化

```c++
template <typename R, typename... Args>
struct function_traits_helper
{
    static constexpr auto param_count = sizeof...(Args);
    using return_type = R;

    template <std::size_t N>
    using param_type = std::tuple_element_t<N, std::tuple<Args...>>;
};

// int(*)(int, int)
template <typename R, typename... Args>
struct function_traits<R(*)(Args...)> : public function_traits_helper<R, Args...>
{
};

// int(&)(int, int)
template <typename R, typename... Args>
struct function_traits<R(&)(Args...)> : public function_traits_helper<R, Args...>
{
};

// int(int, int)
template <typename R, typename... Args>
struct function_traits<R(Args...)> : public function_traits_helper<R, Args...>
{
};
```



## 针对lambda进行偏特化

假设在main函数中定义一个匿名函数lambda，通过模板参数类型推导判断它的类型

```c++
template <typename Func>
void traits_test(Func&&) {
    
}

int main()
{
    auto f = [](int a, int b) { return a + b; };
    // Func = main()::<lambda(int, int)> const
    // Func::operator() = int(main()::<lambda(int, int)>*)(int, int) const
    traits_test(f);
    return 0;
}
```

lambda实际上是一个匿名函数对象，可以理解为内部也重载了operator()函数，所以如果将lambda整体进行推导，那么会推导出一个main()::<lambda(...)>类型，为了获取元数据，需要将它的operator()展开

特化版本

```c++
template <typename R, typename... Args>
struct function_traits_helper
{
    static constexpr auto param_count = sizeof...(Args);
    using return_type = R;

    template <std::size_t N>
    using param_type = std::tuple_element_t<N, std::tuple<Args...>>;
};

template <typename ClassType, typename R, typename... Args>
struct function_traits<R(ClassType::*)(Args...) const> : public function_traits_helper<R, Args...>
{
    using class_type = ClassType;
};

template <typename T>
struct function_traits : public function_traits<decltype(&T::operator())> {};
```

> 通过std::function和std::bind绑定的函数对象和lambda类型相同，不需要额外的特化

## 增加const版本

针对某些可能推导出const的增加const版本，比如上述的lambda

------

完整代码请[参考这里](https://github.com/rocwangp/code_part/blob/master/funciton_traits.cc)

