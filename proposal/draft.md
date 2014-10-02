Strongly Typed Bitset
==========================================

* Document Number: NXXXX
* Date: 2014-08-20
* Programming Language C++, Library Evolution Working Group
* Reply-to: Matthew Fioravante <fmatthew5876@gmail.com>

The latest draft, reference header, and links to past discussions on github: 

* <https://github.com/fmatthew5876/stdcxx-bitset>

Introduction
=============================

The proposal enhances `std::bitset` by adding a new template parameter which 
allows the programmer to control the size and alignment of the underlying representation.

This proposal is aimed for a C++ Technical Specification.

Impact on the standard
=============================

This proposal is a pure library extension. It changes the
template signature of `std::bitset` by adding a new template parameter.
It also adds 2 new aliases `std::fast_bitset` and `std::small_bitset`.

Impact on Implementations
=============================

Implementation of this proposal will likely break the ABI of all standard library
implementations of `std::bitset`. Adding a new template parameter may change the mangled
name of the type.

Motivation and Design
================

When dealing with bits, the classical method of implementation is to use
an unsigned integral type and perform bit operations using logical operations, shifts, and masks.

With C++11, we have `std::bitset` which is a nice abstraction for acting on the individual
bits of a fixed size quantity.  The objective
of `std::bitset` should be to entirely replace unsigned integers for implementing small collections of
bit flags. 

Often times, sets of bits or flags are included in binary protocols or
other tightly packed small objects in memory. Due to the precise
size and alignment requirements of such domains, `std::bitset` is
unusable because the programmer has no control over the alignment or
size of `std::bitset`. Indeed it appears that `std::bitset` on
x86_64 linux gcc is always at least 8 bytes.
In some situations, the
overhead of the extra space makes bitset unusable.

In addition, sometimes users may want a `bitset` object whose underlying representation
has been automatically optimize for speed or space on a given platform. For this we provide
`fast_bitset` and `small_bitset`.

Technical Specification
====================

We will now describe the additions to the `<experimental/bitset>` header.

std::bitset&lt;N,T&gt;
--------------------------

This new declaration is aimed to eventually replace the current `std::bitset`.

    namespace std {
    namespace experimental {
    template <size_t N, typename T = /* Implementation defined */>
      class bitset;
    } //namespace experimental
    } // namespace std

`std::bitset<N,T>` shall have the following constraints:

    static_assert(is_integral<T>::value);
    static_assert(alignof(bitset<N,T>) == alignof(T[N / (sizeof(T) * CHAR_BIT) + 1]));
    static_assert(sizeof(bitset<N,T>) == sizeof(T[N / (sizeof(T) * CHAR_BIT) + 1]));

*[note-- `bitset<N,T>` is not required to actually be implemented using an array of type `T`, it only needs to satisfy the size and alignment constraints. --end node]*

There are no changes or additions to any of the public members of `bitset`.

###Implementation guidance for the default `T`

The default type `T` used for `std::bitset<N,T>` is implementation defined. This behavior is provided
for backwards source compatibility with C++14. We offer the following recommendations for
platforms in choosing which type to use.

* Most general platforms should choose `T` as to optimize for speed (see below: `fast_bitset<N>`)
* Memory constrained platforms should choose `T` as to optimize for space (see below: `small_bitset<N>`)
* If backwards compatibility is of primary concern, platforms can continue to use the same type used in C++14.
Note that changing the template arguments of bitset will already break ABI compatibility on most platforms.
Implementors may choose to take this oportunity to change `T` to optimize for speed or space.

### Conceptual implementation

A conforming implementation could be constructed using the following data model.

    template <size_t N, typename T>
      struct alignas(T) bitset {
        T x[N / (sizeof(T) * CHAR_BIT) + 1];
      };

Note that this implementation also works on Linux i386 where `sizeof(long long) == 8` but `struct t { long long x; }; alignof(t) == 4`.

std::fast_bitset<N>
-------------------------

The type `fast_bitset<N>` shall be an alias for `std::bitset<N,T>`, where `T` is choosen to be the fastest underlying representation for `N` bits.

###Example implementation

    template <size_t N>
    struct __fast_bitset_type : public
      std::conditional<N > 32, uint_fast64_t,
        std::conditional<N > 16, uint_fast32_t,
          std::conditional<N > 8, uint_fast16_t,
            uint_fast8_t
          >
        >
      > {};

    template <size_t N> using fast_bitset = bitset<N,typename __fast_bitset_type<N>::type>;

std::small_bitset<N>
-----------------------

The type `fast_bitset<N>` shall be an alias for `std::bitset<N,T>`, where `T` is choosen to be the smallest underlying representation for `N` bits.
Note that there are no restrictions on alignment.

###Example implementation

    template <size_t N>
    struct __small_bitset_type : public
      std::conditional<N % 64 > 32 || N % 64 == 0, uint_least64_t,
        std::conditional<N % 64 > 16, uint_least32_t,
          std::conditional<N % 64 > 8, uint_least16_t,
            uint_least8_t
          >
        >
      > {};

    template <size_t N> using small_bitset = bitset<N,typename __small_bitset_type<N>::type>;

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
