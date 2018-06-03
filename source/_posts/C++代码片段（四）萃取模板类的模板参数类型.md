---
title: C++代码片段（四）萃取模板类的模板参数类型
date: 2018-06-03
categories: C++
tags: C++
---

例如有类型

```c++
Test<int, double, std::string>
```

可以萃取出模板参数分别是

```c++
int, double, std::string
```

方法如下

```c++
#include <tuple>
#include <iostream>

template <typename...>
struct template_argument_type_traits {};

// 因为ClassType是一个模板类，所以用模板的模板参数表示它，即
// template <typename...> class ClassType 表示ClassType是一个模板类
// 这样才可以在特化的时候为它添加模板参数Args...
template <template <typename...> class ClassType, typename... Args>
struct template_argument_type_traits<ClassType<Args...>>
{
    template <std::size_t N>
    using param_type = std::tuple_element_t<N, std::tuple<Args...>>;
};
```
