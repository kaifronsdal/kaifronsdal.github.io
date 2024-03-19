---
layout: post
title: I wrote a faster c++ sorting algorithm
---

These days it’s a pretty bold claim if you say that you invented a sorting algorithm that’s 10% faster than state-of-the-art. You'll quickly find that the advice on every stackexchange post asking for the "fastest sorting algorithm" is always to just use `std::sort`. Recently, with C++17 introducting support for parallelism, sorting performance has skyrocketed by running on all available cores. It is not hard to find algorithms that achieve 10x performance by using [SIMD parrallelism](https://opensource.googleblog.com/2022/06/Vectorized%20and%20performance%20portable%20Quicksort.html) or 19x by using [multiple cores](https://duvanenko.tech.blog/2023/10/29/sorting-19x-faster-than-c-parallel-sort/).

However, all of these optimizations are based around ``std::sort`` which is designed to only work for `std::random_access_iterator` and is not designed to work for `std::forward_iterator`. This is a problem because many of the most popular data structures in C++ are not random access. For example, `std::list` is a doubly linked list and `std::forward_list` is a singly linked list. Instead, these datastructures implement their own sorting algorithms, but they tend to be bandaid patches to the standard merge sort to make the linked lists appear more like random access iterators. We will explore the current implementation of `list::sort` and then propose a few optimizations to make it faster.

## ``list::sort`` Implementation
Below is a current implementation of `list::sort` from the [LLVM Project](https://github.com/llvm/llvm-project/blob/2c2d291b4568381999442e47fc77f949f19be0bc/libcxx/include/list#L1591):

```cpp
template <class _Tp, class _Alloc>
template <class _Comp>
inline
void
list<_Tp, _Alloc>::sort(_Comp __comp)
{
    __sort(begin(), end(), base::__sz(), __comp);
}

template <class _Tp, class _Alloc>
template <class _Comp>
typename list<_Tp, _Alloc>::iterator
list<_Tp, _Alloc>::__sort(iterator __f1, iterator __e2, size_type __n, _Comp& __comp)
{
    switch (__n)
    {
    case 0:
    case 1:
        return __f1;
    case 2:
        if (__comp(*--__e2, *__f1))
        {
            __link_pointer __f = __e2.__ptr_;
            base::__unlink_nodes(__f, __f);
            __link_nodes(__f1.__ptr_, __f, __f);
            return __e2;
        }
        return __f1;
    }
    size_type __n2 = __n / 2;
    iterator __e1 = _VSTD::next(__f1, __n2);
    iterator  __r = __f1 = __sort(__f1, __e1, __n2, __comp);
    iterator __f2 = __e1 = __sort(__e1, __e2, __n - __n2, __comp);
    if (__comp(*__f2, *__f1))
    {
        iterator __m2 = _VSTD::next(__f2);
        for (; __m2 != __e2 && __comp(*__m2, *__f1); ++__m2)
            ;
        __link_pointer __f = __f2.__ptr_;
        __link_pointer __l = __m2.__ptr_->__prev_;
        __r = __f2;
        __e1 = __f2 = __m2;
        base::__unlink_nodes(__f, __l);
        __m2 = _VSTD::next(__f1);
        __link_nodes(__f1.__ptr_, __f, __l);
        __f1 = __m2;
    }
    else
        ++__f1;
    while (__f1 != __e1 && __f2 != __e2)
    {
        if (__comp(*__f2, *__f1))
        {
            iterator __m2 = _VSTD::next(__f2);
            for (; __m2 != __e2 && __comp(*__m2, *__f1); ++__m2)
                ;
            __link_pointer __f = __f2.__ptr_;
            __link_pointer __l = __m2.__ptr_->__prev_;
            if (__e1 == __f2)
                __e1 = __m2;
            __f2 = __m2;
            base::__unlink_nodes(__f, __l);
            __m2 = _VSTD::next(__f1);
            __link_nodes(__f1.__ptr_, __f, __l);
            __f1 = __m2;
        }
        else
            ++__f1;
    }
    return __r;
}
```

This is a lot of code for a simple sorting algorithm The algorithm is a merge sort, but it is implemented in a way that is not very readable. So below is a translation of the algorithm into a more readable form:

```cpp
template <class Tp, class Alloc>
template <class Comp>
typename list<Tp, Alloc>::iterator
list<Tp, Alloc>::sort(iterator begin, iterator end, size_type n, Comp& comp)
{
    // base cases
    if (n <= 1)
        return begin;
    if (n == 2)
    {
        // if the second element is before the first element
        if (comp(*--end, *begin))
        {
            // swap the two elements
            splice(begin, this, end);
            return end;
        }
        return begin;
    }

    // split the list in half and sort each half
    iterator left_end = std::next(begin, n / 2);
    iterator rtrn = left_begin = sort(begin, left_end, n / 2, comp);
    iterator right_begin = left_end = sort(left_end, end, n - (n / 2), comp);

    // merge the two halves
    // if the right half is less than the left half
    if (comp(*right_begin, *left_begin))
    {
        // find the first element in the right half that is greater than the left half
        auto it = std::next(right_begin);
        for (; it != end && comp(*it, *left_begin); ++it)
            ;
        // and insert that range in front of the left half
        splice(left_begin, this, right_begin, it);
        // update the return value
        rtrn = right_begin;
        // update the right half
        right_begin = it;
    }
    else
        // if the right half is greater than the left half
        ++left_begin;
    
    // repeat until one of the halves is empty
    // the only difference to above is that we don't update the return value
    while (left_begin != left_end && right_begin != end)
    {
        if (comp(*right_begin, *left_begin))
        {
            auto it = std::next(right_begin);
            for (; it != end && comp(*it, *left_begin); ++it)
                ;
            splice(left_begin, this, right_begin, it);
            right_begin = it;
        }
        else
            ++left_begin;
    }
    return rtrn;
}
```
In the above human-readable translation we use better naming conventions and replace the node swapping logic with the `splice` function which under the hood is almost identical to the node swapping logic in the oroginal implementation.

This code is much better than before, but it is still not very readable. The underlying algorithm is a merge sort which consists of two parts: splitting the list in half and merging the two halves. Using a bit of function decomposition yeilds the following code (`...` is used to indicate the parts of the code that are unchanged):

```cpp
template <class Tp, class Alloc>
template <class Comp>
typename list<Tp, Alloc>::iterator
list<Tp, Alloc>::merge(iterator left_begin, iterator left_end, iterator right_begin, iterator right_end, Comp& comp)
{   
    iterator rtrn = left_begin;
    // if the right half is less than the left half
    if (comp(*right_begin, *left_begin))
    {
        // ...

    // repeat until one of the halves is empty
    while (left_begin != left_end && right_begin != right_end)
    {
        // ...
    return rtrn;
}

template <class Tp, class Alloc>
template <class Comp>
typename list<Tp, Alloc>::iterator
list<Tp, Alloc>::sort(iterator begin, iterator end, size_type n, Comp& comp)
{
    // base cases
    if (n <= 1)
        return begin;
    if (n == 2)
    {
        // ...

    // split the list in half and sort each half
    iterator left_end = std::next(begin, n / 2);
    iterator rtrn = left_begin = sort(begin, left_end, n / 2, comp);
    iterator right_begin = left_end = sort(left_end, end, n - (n / 2), comp);

    // merge the two halves
    return merge(left_begin, left_end, right_begin, end, comp);
}
```
Ah, much better!

## Optimizations

Because we are using a linked list, we can't use the same optimizations that we would use for an array. In particular, we don't have random access. For most of the algorithm we don't need it. For example, when merging we simply iterate through the lists in order. However, when we split the list in half we need to iterate through the list to find the middle.
```cpp
iterator left_end = std::next(begin, n / 2);
```
This is a very wasteful operation. Even though it is $O(n)$ and doesn't not effect the asymtopic complexity of the algorithm, the effect is very noticable. Modern hardware is designed for sequential access (through various caching, prefetching mechanisms, and speculative execution; for a more in-depth explanation see [TODO]). This means that iterating through a linked list is much slower than iterating through an array. So on top of the $O(n)$ complexity fo rthe `std::next` function, we have the additional downside of not being able to take advantage of modern hardwares sequential access optimizations.

The key insight is that during the sorting process we are going to be iterating through the list anyways. Thus, if we have sort return the last element of the sorted list, we can avoid the $O(n)$ operation of finding the middle of the list. Additionally, because we already pass in the size of the list, we don't need to pass in the last element either. 

```cpp
template <class Tp, class Alloc>
template <class Comp>
typename list<Tp, Alloc>::iterator
iterator merge(iterator left_begin, iterator left_end, iterator right_begin, iterator right_end, Comp& comp)

template <class Tp, class Alloc>
template <class Comp>
typename list<Tp, Alloc>::iterator
iterator sort(iterator begin, size_type n, Comp& comp)
{
    // base cases
    if (n <= 1)
        return begin;
    if (n == 2)
    {
        // ...
    
    iterator left_end = sort(begin, n / 2, comp);
    iterator right_begin = ++left_end;
    iterator right_end = sort(right_begin, n - (n / 2), comp);
    merge(begin, left_end, right_begin, right_end, comp);
    return right_end;
}
```