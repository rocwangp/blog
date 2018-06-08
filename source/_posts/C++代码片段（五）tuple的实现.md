---
title: C++代码片段（五）tuple的实现
date: 2018-06-08
categories: C++
tags: C++
---





元组tuple是多类型的集合，可以当做升级版的pair来用。因为tuple可以保存多个任意类型的值，导致不能够在一个类里面保存所有的数据，所以采用每一层保存一个数据的方法实现

<!--more-->

## 实现tuple

```c++
template <typename... Args>
class tuple
{
};

/* 每一层保存一个T类型的数据 */
template <typename T, typename... Args>
class tuple<T, Args...> : public tuple<Args...>
{
    public:
        tuple() = default;

        tuple(T&& t, Args&&... args)
            : tuple<Args...>(std::forward<Args>(args)...),
              value_(std::forward<T>(t))
        { }

        template <std::size_t N>
        decltype(auto) get() {
            static_assert(N<=sizeof...(Args));
            if constexpr (N == 0) {
                return (value_);
            }
            else {
                return (this->tuple<Args...>::template get<N - 1>());
            }
        }

        template <std::size_t N>
        decltype(auto) get() const {
            static_assert(N<=sizeof...(Args));
            if constexpr (N == 0) {
                return (value_);
            }
            else {
                return (this->tuple<Args...>::template get<N - 1>());
            }
        }
    protected:
        T value_;
};
```
## 实现tuple_element

tuple_element&lt;N, Tuple&gt;萃取元组Tuple的第N个类型。依次遍历每一个模板参数，直到遍历到第N个后返回，采用继承的方式保证最外层的tuple_element_helper可以获取最终的结果

模板的模板参数用来偏特化元组类型，目的是萃取出参数列表

```c++
template <std::size_t N, template <typename...> class Tuple, typename... ArgsWrapper>
struct tuple_element<N, Tuple<ArgsWrapper...>>
{
    template <std::size_t M, typename T, typename... Args>
    struct tuple_element_helper : public tuple_element_helper<M-1, Args..., void>
    {
    };

    template <typename T, typename... Args>
    struct tuple_element_helper<0, T, Args...>
    {
        using type = T;
    };

    using type = typename tuple_element_helper<N, ArgsWrapper..., void>::type;
};

template <std::size_t N, typename Tuple>
using tuple_element_t = typename tuple_element<N, Tuple>::type;
```

## 实现tuple_size

tuple_size&lt;Tuple&gt;萃取元组Tuple中包含的数据个数。直接利用模板的模板参数萃取参数列表，然后使用sizeof计算参数列表的个数即可

```c++
template <typename Tuple>
struct tuple_size : public std::integral_constant<std::size_t, 0>
{
};

template <template <typename...> class Tuple, typename... Args>
struct tuple_size<Tuple<Args...>> : public std::integral_constant<std::size_t, sizeof...(Args)>
{
};

template <typename Tuple>
static constexpr inline auto tuple_size_v = tuple_size<Tuple>::value;
```

## 实现make_tuple

make_tuple根据参数创建tuple，引用cppreference上面的标准介绍

> 创建 tuple 对象，从参数类型推导目标类型。
>
> 对于每个 `Types...` 中的 `Ti` ， `Vtypes...` 中的对应类型 `Vi` 为 [std::decay](http://zh.cppreference.com/w/cpp/types/decay)<Ti>::type ，除非应用 [std::decay](http://zh.cppreference.com/w/cpp/types/decay) 对某些类型 `X` 导致 [std::reference_wrapper](http://zh.cppreference.com/w/cpp/utility/functional/reference_wrapper)<X> ，该情况下推导的类型为 `X&` 。

decay会移除类型的const和volatile限定符以及引用，将左值转换成右值，但是对于reference_wrapper类型需要特殊处理，保证tuple的模板参数也同样是引用包装器

```c++
namespace detail
{
    template <typename T>
    struct unwrap_refwrapper
    {
        using type = T;
    };

    template <typename T>
    struct unwrap_refwrapper<std::reference_wrapper<T>>
    {
        using type = T&;
    };

    template <typename T>
    using special_decay_t = typename unwrap_refwrapper<std::decay_t<T>>::type;
}

template <typename... Args>
tuple<Args...> make_tuple(Args&&... args) {
    return tuple<detail::special_decay_t<Args>...>(std::forward<Args>(args)...);
}
```



## 实现tuple_cat

tuple_cat用于将两个tuple连接起来，依次将每个tuple的数据取出放到可变模板参数列表中，最后重新组装即可

```c++
namespace detail
{
    template <std::size_t N1, std::size_t M1, std::size_t N2, std::size_t M2, typename Tuple1, typename Tuple2, typename... Args>
    auto tuple_cat_impl(Tuple1&& t1, Tuple2&& t2, Args&&... args) {
        if constexpr (N1 == M1 && N2 == M2) {
            return make_tuple(std::forward<Args>(args)...);
        }
        else if constexpr (N1 == M1) {
            return tuple_cat_impl<N1, M1, N2+1, M2>(std::forward<Tuple1>(t1),
                                                       std::forward<Tuple2>(t2),
                                                       std::forward<Args>(args)...,
                                                       t2.template get<N2>());
        }
        else {
            return tuple_cat_impl<N1+1, M1, N2, M2>(std::forward<Tuple1>(t1),
                                                       std::forward<Tuple2>(t2),
                                                       std::forward<Args>(args)...,
                                                       t1.template get<N1>());
        }
    }
}
template <typename... Args1, typename... Args2>
auto tuple_cat(const tuple<Args1...>& t1, const tuple<Args2...>& t2) {
    return detail::tuple_cat_impl<0, sizeof...(Args1), 0, sizeof...(Args2)>(t1, t2);
}
```

