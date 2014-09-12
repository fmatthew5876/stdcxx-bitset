Size and Alignment control for bitset
==========================================

* Document Number: NXXXX
* Date: 2014-08-20
* Programming Language C++, Library Working Group
* Reply-to: Matthew Fioravante <fmatthew5876@gmail.com>

The latest draft, reference header, and links to past discussions on github: 

* <https://github.com/fmatthew5876/stdcxx-bitset>

Introduction
=============================

The proposal enhances `std::bitset` by adding a new template parameter which 
allows the programmer to control the size and alignment of the underlying representation.
It also provides guidance to implementors for how to implement `std::bitset<N>`.

This proposal is aimed for a C++ Technical Specification.

Impact on the standard
=============================

This proposal is a pure library extension. It changes the
template signature of `std::bitset` by adding a new template parameter.
It also adds implementation recommendations to the standard for `std::bitset`.

Impact on Implementations
=============================

Implementation of this proposal will likely break the ABI of all standard library
implementations of `std::bitset`. Adding a new template parameter may change the mangled
name of the type. After following the recommendations laid out in this paper, some
implementations may need to change the ABI of the pre-existing `std::bitset<N>`.

Motivation and Design
================

When dealing with bits, the usual method of implementation is to use
an `unsigned` integral type and perform bit operations using logical operators, shifts, and masks.

With C++11, we have `std::bitset` which is a nice abstraction for acting on the individual
bits of a fixed size quantity.  The objective
of `std::bitset` is to entirely replace unsigned integers for implementing small collections of
bit flags. 

Often times, sets of bits or flags are included in binary protocols or
other tightly packed small objects in memory. Due to the precision
size and alignment requirements of such domains, `std::bitset` is
unusable because the programmer has no control over the alignment or
size of `std::bitset`.

There is no guidance to implementors on how they should implement `std::bitset`. There
are two competing goals of optimizing for speed or for space. We argue that implementions
should always optimize for speed by default. Users who have tighter size contraints can
specify them through the interface.

Technical Specification
====================

We will now describe the additions to the `<experimental/bitset>` header.
This new declaration is aimed to eventually the current `std::bitset`.

    namespace std {
    namespace experimental {
    template <size_T N, typename T = /* Implementation defined */>
      class bitset;
    } //namespace experimental
    } // namespace std

`std::bitset<N,T>` shall have the following constraints:

    static_assert(alignof(bitset<N,T>) == alignof(T));
    static_assert(sizeof(bitset<N,T>) == sizeof(T) * (N / (sizeof(T) * CHAR_BIT) + 1));

*note-- `bitset<N,T>` is not required to be implemented using type `T`, it only needs to satisfy the size and alignment constraints. --end node*

There are no changes or additions to any of the public members of `bitset`.

Implementors guide for std::bitset&lt;N&gt;
====================================

The following paragraph shall be included in the standard:

------------------------------

When choosing which type to use for `std::bitset<N>`, implementors should first try to optimize for
*speed*, and then optimize for *size*. A good choice of underlying type might be `uintN_fast_t` for each
respective size category. Implementations are encouraged to use simd types for larger bitsets where appropriate.

------------------------------

We recommend implementations to optimize for speed over space for the following reasons:

* Optimizing for speed is the most commonly desired default for majority of use cases.
* Knowing and choosing the optimal integer size for speed requires platform dependent knowledge.
* Optimizing for speed is a very simple objective with one variable
* Optimizing for space requires programmer input as there are 2 variables in play, alignment and size

## Conceptual implementation

A conforming implementation could be constructed using the following data model.

    template <size_t N, typename T>
      struct bitset {
        T x[sizeof(T) * (N / (sizeof(T) * CHAR_BIT) + 1)];
      } alignas(T);

Note that this implementation also works on Linux i386 where `sizeof(long long) == 8` but `struct { long long t; } t; sizeof(t) == 4`.

Use Cases
==================

* Binary protocols
* CPU emulation, modeling instruction sets with flags
* Tightly packing data into small blocks of memory.
* Data Polymorphism (as opposed to behavior polymorphism using virtual)

Acknowledgments
====================

* Thank you to everyone one the std proposals forum.


<!--
References
==================

* <a name="N3864"></a>[N3864] Fioravante, Matthew *N3864 - A constexpr bitwise operations library for C++*, Available online at <https://github.com/fmatthew5876/stdcxx-bitops>
* <a name="LXR"></a>[LXR] *Linux/include/linux/kernel.h* Available online at <http://lxr.free-electrons.com/source/include/linux/kernel.h#L50>
* <a name="IsoCpp"></a>[IsoCpp] *ISO C++ standard*
* <a name="clang"></a> *"clang" C Language Family Frontend for LLVM* Available online at <http://clang.llvm.org/>
-->
