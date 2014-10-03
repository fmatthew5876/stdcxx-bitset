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
It also adds 2 new aliases `std::fast_bitset` and `std::small_bitset` and
3 new public members, a new constructor overload and a new assignment operator overload.

Impact on Implementations
=============================

Implementation of this proposal will likely break the ABI of all standard library
implementations of `std::bitset`. Adding a new template parameter may change the mangled
name of the type. If accepted, the deployment date of this proposal into the standard should
be carefully considered to avoid too many ABI breaks.

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
x86\_64 linux gcc is always at least 8 bytes.
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
    static_assert(alignof(bitset<N,T>) == alignof(T[N / (sizeof(T) * CHAR_BIT) + (n % (sizeof(T) * CHAR_BIT) != 0)];
    static_assert(sizeof(bitset<N,T>) == sizeof(T[N / (sizeof(T) * CHAR_BIT) + (n % (sizeof(T) * CHAR_BIT) != 0)];

*[note-- `bitset<N,T>` is not required to actually be implemented using an array of type `T`, it only needs to satisfy the size and alignment constraints. --end node]*

###Implementation guidance for the default `T`

The default type `T` used for `std::bitset<N,T>` is implementation defined. This behavior is provided
for backwards source compatibility with C++14. We offer the following recommendations for
platforms in choosing which type to use.

* Most general platforms should choose `T` as to optimize for speed (see below: `fast_bitset<N>`)
* Memory constrained platforms should choose `T` as to optimize for space (see below: `small_bitset<N>`)
* If backwards compatibility is of primary concern, platforms can continue to use the same type used in C++14.
Note that changing the template arguments of bitset will already break ABI compatibility on most platforms.
Implementors may choose to take this oportunity to change default `T` to optimize for speed or space.

### Conceptual implementation

A conforming implementation could be constructed using the following data model.

    template <size_t N, typename T>
      struct alignas(T) bitset {
        T x[N / (sizeof(T) * CHAR_BIT) + (n % (sizeof(T) * CHAR_BIT) != 0)];
      };

Note that this implementation also works on Linux i386 where `sizeof(long long) == 8` but `struct t { long long x; }; alignof(t) == 4`.

Accessing the underlying storage
--------------------------------

We propose to add the following public members to `std::bitset`:

    template <size_t N, typename T>
      using bitset<N,T>::underlying_type = std::array<T, N / (sizeof(T) * CHAR_BIT) + (n % (sizeof(T) * CHAR_BIT) != 0)>;

The `underlying_type` typedef is present to inform the programmer what kind of storage is being used by the `bitset`.
The `bitset` must have the same size and alignment as this type.

    template <size_t N, typename T>
      underlying_type& biset<N,T>::underlying() noexcept;
    template <size_t N, typename T>
      const underlying_type& bitset<N,T>::underlying() const noexcept;

These 2 accessors allow the programmer manual access to the underlying storage.
The values of the bits from `0` to `N-1` in the returned reference will be the same as the corresponding bits in the bitset.
The values of the bits `>= N` in the returned reference are implementation defined.

Any modifications performed on bits `0` to `N-1`
of the returned reference will also be performed on the `bitset` object. Likewise modifying the bits in the bitset
will also modify the bits of the reference.
Modifying any bits `>= N` will have no effect on the underlying bitset and their state is not guaranteed to remain
consistent after calling any member function of `bitset` or any free function which modifies `bitset`.


Note that while we do not require implementations to actually use `underlying_type` to implement `bitset`, we do require them to
match the size and alignment constraints of `underlying_type`. Therefore if bitset is implemented using a different type, the
implementation can internally perform a cast to `underlying_type` to implement these methods.

Copying bitsets of different types.
-------------------------

A bitset using one representation `T` may be copied from another bitset using a different representation `U`. We add the following
constructor and assignment operator overloads.

    template <size_t N, typename T> template <typename U>
      bitset<N,T>::bitset(const bitset<N,U>& other);
    template <size_t N, typename T> template <typename U>
      bitset<N,T>& bitset<N,T>::operator=(const bitset<N,U>& other);


std::fast\_bitset&lt;N&gt;
-------------------------

The type `fast_bitset<N>` shall be an alias for `std::bitset<N,T>`, where `T` is choosen to be the fastest underlying representation for `N` bits.

###Possible implementation

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

std::small\_bitset&lt;N&gt;
-----------------------

The type `fast_bitset<N>` shall be an alias for `std::bitset<N,T>`, where `T` is choosen to be the smallest underlying representation for `N` bits.
Note that there are no restrictions on alignment.

###Possible implementation

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
* Bit flags

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
