# Fixing allocator usage for `operator+(basic_string)`

| []()  | []() |
|---|---|
| Paper number | P1165R0 |
| Reply-to | Tim Song &lt;rs2740@gmail.com&gt; |
| Audience | LWG |

## Abstract

Allocator propagation for `basic_string`'s `operator+` is haphazard, inconsistent, and a source of implementation divergence. Let's make it consistent.

## Introduction

Let `lhs` and `rhs` be two `basic_string` lvalues with a `char` `value_type`. The following table shows the allocator used for the result for each of the 12 `operator+` overloads, in the working paper and three major implementations:

| Expression | WP | libstdc++ (trunk) | libc++ (trunk) | MSVC STL (19.00.23506) |
|---|----|----|----|----|
| `lhs + rhs` | ~~SOCCC on~~ SOCCC on `lhs` | SOCCC on `lhs` | `lhs` (no SOCCC) | default-constructed |
| `lhs + std::move(rhs)` | `rhs` | `rhs` | `rhs` | `rhs` |
| `std::move(lhs) + rhs` | `lhs` | `lhs` | `lhs` | `lhs` |
| `std::move(lhs) + std::move(rhs)` | `lhs` ~~"or equivalently" `rhs`~~ | `lhs` | `lhs` | `lhs` |
| `lhs + "str"` | default-constructed | SOCCC on `lhs` | `lhs` | default-constructed |
| `lhs + 'c'` | default-constructed | SOCCC on `lhs` | `lhs` | default-constructed |
| `std::move(lhs) + "str"` | `lhs` | `lhs` | `lhs` | `lhs` |
| `std::move(lhs) + 'c'` | `lhs` | `lhs` | `lhs`| `lhs` |
| `"str" + rhs` | default-constructed | default-constructed | `rhs` | default-constructed |
| `'c' + rhs` | default-constructed | default-constructed | `rhs` | default-constructed |
| `"str" + std::move(rhs)` | `rhs` | `rhs` | `rhs` | `rhs` |
| `'c' + std::move(rhs)` | `rhs` | `rhs` | `rhs` | `rhs` |
<small>(SOCCC == `select_on_container_copy_construction`. The ~~struckthrough~~ text is proposed to be removed by P1148R0.)</small>

This is haphazard, inconsistent, and a source of implementation divergence.

## Discussion

In the discussion of [LWG2402](https://cplusplus.github.io/LWG/issue2402), Pablo Halpern explained the patterns we use for constructors of allocator-aware containers (modifications mine):

> - Every constructor needs a version with and without an allocator argument (possibly through the use of default arguments).
> - Every constructor except the copy [and move] constructor[s] for which an allocator is not provided uses a default-constructed allocator.

Copy constructors obtain the allocator to use via `select_on_container_copy_construction`; move constructors obtain the allocator to use by move constructing from the source's allocator (which is equivalent to copy after [LWG2593](https://wg21.link/lwg2593)).

`operator+` constructs a new string, but it is not a copy or move constructor any more than `basic_string(const basic_string&, size_t, size_t, const Allocator&)`. Its operands are simply sources of characters. It should therefore consistently use a default-constructed allocator.

As a binary operator, it is not possible to add an allocator argument to `operator+`.  Designing a string concatenation API with allocator support is beyond the scope of this paper. Users desiring to control the allocator used for the result should therefore use the member functions of `basic_string` directly.

## Proposed wording

This wording is relative to [N4762](https://wg21.link/n4762).

Replace `[string.op+]` in its entirety with the following:

```c++
template<class charT, class traits, class Allocator>
  basic_string<charT, traits, Allocator>
    operator+(const basic_string<charT, traits, Allocator>& lhs,
              const basic_string<charT, traits, Allocator>& rhs);

template<class charT, class traits, class Allocator>
  basic_string<charT, traits, Allocator>
    operator+(const basic_string<charT, traits, Allocator>& lhs, const charT* rhs);
```

<sup>1</sup> _Effects_: Equivalent to:

```c++
    basic_string<charT, traits, Allocator> r(lhs, Allocator());
    r.append(rhs);
    return r;
```

```c++
template<class charT, class traits, class Allocator>
  basic_string<charT, traits, Allocator>
    operator+(basic_string<charT, traits, Allocator>&& lhs,
              const basic_string<charT, traits, Allocator>& rhs);
template<class charT, class traits, class Allocator>
  basic_string<charT, traits, Allocator>
    operator+(basic_string<charT, traits, Allocator>&& lhs, const charT* rhs);
```

<sup>2</sup> _Effects_: Equivalent to:

```c++
    basic_string<charT, traits, Allocator> r(std::move(lhs), Allocator());
    r.append(rhs);
    return r;
```


```c++
template<class charT, class traits, class Allocator>
  basic_string<charT, traits, Allocator>
    operator+(basic_string<charT, traits, Allocator>&& lhs,
              basic_string<charT, traits, Allocator>&& rhs);
```

<sup>3</sup> _Effects_: Equivalent to:

```c++
    basic_string<charT, traits, Allocator> r(std::move(lhs), Allocator());
    r.append(rhs);
    return r;
```

except that both  `lhs` and `rhs` are left in valid but unspecified states.

> _Drafting note_: The intent is to allow the implementation to move from either operand. &mdash; _end drafting note_

```c++
template<class charT, class traits, class Allocator>
  basic_string<charT, traits, Allocator>
    operator+(const basic_string<charT, traits, Allocator>& lhs,
              basic_string<charT, traits, Allocator>&& rhs);
template<class charT, class traits, class Allocator>
  basic_string<charT, traits, Allocator>
    operator+(const charT* lhs, basic_string<charT, traits, Allocator>&& rhs);
```

<sup>4</sup> Effects: Equivalent to:

```c++
    basic_string<charT, traits, Allocator> r(std::move(rhs), Allocator());
    r.insert(0, lhs);
    return r;
```

```c++
template<class charT, class traits, class Allocator>
  basic_string<charT, traits, Allocator>
    operator+(const charT* lhs, const basic_string<charT, traits, Allocator>& rhs);
```

<sup>5</sup> _Effects_: Equivalent to:

```c++
    basic_string<charT, traits, Allocator> r = lhs;
    r.append(rhs);
    return r;
```

```c++
template<class charT, class traits, class Allocator>
  basic_string<charT, traits, Allocator>
    operator+(charT lhs, const basic_string<charT, traits, Allocator>& rhs);
```

<sup>6</sup> _Effects_: Equivalent to:

```c++
    basic_string<charT, traits, Allocator> r(1, lhs);
    r.append(rhs);
    return r;
```

```c++
template<class charT, class traits, class Allocator>
  basic_string<charT, traits, Allocator>
    operator+(charT lhs, basic_string<charT, traits, Allocator>&& rhs);
```

<sup>7</sup> _Effects_: Equivalent to:

```c++
    basic_string<charT, traits, Allocator> r(std::move(rhs), Allocator());
    r.insert(r.begin(), lhs);
    return r;
```

```c++
template<class charT, class traits, class Allocator>
  basic_string<charT, traits, Allocator>
    operator+(const basic_string<charT, traits, Allocator>& lhs, charT rhs);
```

<sup>8</sup> _Effects_: Equivalent to:

```c++
    basic_string<charT, traits, Allocator> r(lhs, Allocator());
    r.push_back(rhs);
    return r;
```

```c++
template<class charT, class traits, class Allocator>
  basic_string<charT, traits, Allocator>
    operator+(basic_string<charT, traits, Allocator>&& lhs, charT rhs);
```

<sup>9</sup> _Effects_: Equivalent to:

```c++
    basic_string<charT, traits, Allocator> r(std::move(lhs), Allocator());
    r.push_back(rhs);
    return r;
```

<!--
## Appendix

### Testing code

```c++
#include <string>
#include <cstdlib>
#include <new>
#include <iostream>

template <class T>
struct Mallocator {
  typedef T value_type;
  Mallocator() : i(100 * dtag++) { }
  explicit Mallocator(int i) : i(i) {}
  Mallocator(const Mallocator& o) = default;
  template <class U> constexpr Mallocator(const Mallocator<U>& u) noexcept : i(u.i) {}
  [[nodiscard]] T* allocate(std::size_t n) {
    if(n > std::size_t(-1) / sizeof(T)) throw std::bad_alloc();
    if(auto p = static_cast<T*>(std::malloc(n*sizeof(T)))) return p;
    throw std::bad_alloc();
  }
  void deallocate(T* p, std::size_t) noexcept { std::free(p); }
  Mallocator select_on_container_copy_construction() const { return Mallocator(i * 1000); };
  inline static int dtag = 1;
  int i;
};
template <class T, class U>
bool operator==(const Mallocator<T>&, const Mallocator<U>&) { return true; }
template <class T, class U>
bool operator!=(const Mallocator<T>&, const Mallocator<U>&) { return false; }


int main() {
    std::basic_string<char, std::char_traits<char>, Mallocator<char>> l, r;
    #define PALLOC(l, r) std::cout << #l " + "  #r ": "<< (l + r).get_allocator().i << '\n'
    PALLOC(l, r);
    PALLOC(l, std::move(r));
    PALLOC(std::move(l), r);
    PALLOC(std::move(l), std::move(r));
    PALLOC(l, "str");
    PALLOC(l, 'c');
    PALLOC(std::move(l), "str");
    PALLOC(std::move(l), 'c');
    PALLOC("str", r);
    PALLOC('c', r);
    PALLOC("str", std::move(r));
    PALLOC('c', std::move(r));
}
```
-->