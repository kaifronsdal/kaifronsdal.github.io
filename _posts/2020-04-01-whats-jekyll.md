---
layout: post
title: I wrote a faster c++ sorting algorithm
---

These days it’s a pretty bold claim if you say that you invented a sorting algorithm that’s 10% faster than state-of-the-art. You'll quickly find that the advice on every stackexchange post asking for the "fastest sorting algorithm" is always to just use `std::sort`. Recently, with C++17 introducting support for parallelism, sorting performance has skyrocketed by running on all available cores. It is not hard to find algorithms that achieve 10x performance by using [SIMD parrallelism](https://opensource.googleblog.com/2022/06/Vectorized%20and%20performance%20portable%20Quicksort.html) or 19x by using [multiple cores](https://duvanenko.tech.blog/2023/10/29/sorting-19x-faster-than-c-parallel-sort/).

However, all of these optimizations are based around ``std::sort`` which is designed to only work for `std::random_access_iterator` and is not designed to work for `std::forward_iterator`. This is a problem because many of the most popular data structures in C++ are not random access. For example, `std::list` is a doubly linked list and `std::forward_list` is a singly linked list. Instead, these datastructures implement their own sorting algorithms, but they tend to be bandaid patches to the standard merge sort to make the linked lists appear more like random access iterators.

## ``list::sort``
Below is a current implementation of `std::list::sort` from the [LLVM Project](https://github.com/llvm/llvm-project/blob/2c2d291b4568381999442e47fc77f949f19be0bc/libcxx/include/list#L1591):

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
    iterator left_end = STD::next(begin, n / 2);
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
    
    // merge the two halves
    while (left_begin != left_end && right_begin != end)
    {
        // if the right half is less than the left half
        if (comp(*right_begin, *left_begin))
        {
            // find the first element in the right half that is greater than the left half
            auto it = std::next(right_begin);
            for (; it != end && comp(*it, *left_begin); ++it)
                ;
            // and insert that range in front of the left half
            splice(left_begin, this, right_begin, it);
            // update the right half
            right_begin = it;
        }
        else
            // if the right half is greater than the left half, look at the next element in the left half
            ++left_begin;
    }
    return rtrn;
}
    
```