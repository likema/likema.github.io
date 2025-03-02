---
title: std::forward_list 简介
date: 2021-12-03 23:35:46
categories: [C++]
---

[`std::forward_list`](https://en.cppreference.com/w/cpp/container/forward_list) 为 C++ 11 STL 的 **单向** 链表:

* 仅记录了首指针，未记录尾指针和链表长度，即无 `push_back` 和 `size` 方法。
* 相比 [`std::list`](https://en.cppreference.com/w/cpp/container/list) （双向链表，以下简化为 `list`）
    * 节约内存空间
    * 仅能 **顺序访问** ，无法 **倒序访问**
    * `forward_list::sort` 时间复杂度 **O(n log n)** ，但效率不如 `list::sort` ，尤其链表包含大量元素时。

因 `forward_list` 与 `list` 大部分方法相同，这里仅介绍 `forward_list` 不具备的能力。

## 顺序插入

因无 `forward_list::push_back` 方法，顺序插入 `forward_list` 一组元素得依靠调用者 **自行** 保存尾指针。


```cpp
forward_list<int> fl;
auto iter = fl.before_begin();

for (int i = 0; i < 9; ++i) {
	iter = fl.insert_after(iter, i);
}
```

[`before_begin`](https://en.cppreference.com/w/cpp/container/forward_list/before_begin) 返回首元素之前的元素（placeholder）。该元素实际不存在，访问它将导致未定义行为。

## 顺序复制

* `back_inserter` 返回的 `std::back_insert_iterator` 要求容器实现了 `push_back` 方法。
* `inserter` 返回的 `std::insert_iterator` 要求容器实现了 `insert` 方法。

显然，两者皆不适用于 `forward_list` ，然 `inserter` 最接近：


```cpp
forward_list<int> fl;
vector<int> v{1, 2, 3, 4, 5, 6};

copy(v.begin(), v.end(), inserter(fl, fl.before_begin()));
```

为使上述代码能被编译和运行，须偏特化 `std::insert_iterator` :

```cpp
namespace std {

template <typename T>
class insert_iterator<forward_list<T> >:
    public iterator<output_iterator_tag, void, void, void, void>
{
public:
    typedef forward_list<T> container_type;
    typedef typename container_type::iterator iterator_type;

    insert_iterator(container_type& c, iterator_type i):
        container_(c), iter_(i)
    {
    }

    insert_iterator& operator = (const typename container_type::value_type& v)
    {
        iter_ = container_.insert_after(iter_, v);
        return *this;
    }

    insert_iterator& operator = (const typename container_type::value_type&& v)
    {
        iter_ = container_.insert_after(iter_, std::move(v));
        return *this;
    }

    insert_iterator& operator * () { return *this; }
    insert_iterator& operator ++ () { return *this; }
    insert_iterator& operator ++ (int) { return *this; }
protected:
    container_type& container_;
    iterator_type iter_;
};

} // namespace std
```

此法不支持让用户定义 `forward_list` 的 allocator 。

完美办法为重新实现 `forward_list_insert_iterator` 和 `forward_list_inserter` 。

## 求长度

因 `forward_list` 未保存长度——无 `size` 方法，故须 **O(n)** 时间复杂度的 `std::distance` 计算长度：

```cpp
forward_list<int> fl{1, 2, 3, 4, 5, 6};
auto size = std::distance(fl.begin(), fl.end());
```

而 `forward_list::empty` （长度是否为 `0` ）的时间复杂度 **O(1)**

值得一提 [`boost::slist`](https://www.boost.org/doc/libs/1_77_0/doc/html/boost/container/slist.html) 高度相似于 `forward_list` ，但前者保存长度，即 `boost::slist::size` 方法时间复杂度 **O(1)**
