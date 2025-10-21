+++
title = "Type safe packed pointers in C"
description = "I beat C"
date = "2025-10-20"
+++

## Intro

Imagine you have allocator that works on top of huge contagious memory region. For instance because you know upper-bound of memory you will allocate. This maybe an arena allocator, slab allocator or regular `malloc`-like allocator, does not matter.

Usually allocators return pointers that consume 8 bytes on 64-bit platforms. There is a way to reduce this number by a factor and
preserve type-safety with zero memory foot-print.

Along the lines I will explain step-by-step how I came up with this approach and will rant about C. In case you don't interested, jump straight to the [end](#full-code) where you will find complete example and links.

## Indexes

The first thing that comes to mind is indexes. Let's say our allocator works on top region `[N, M)`. Then if `M - N + 1` is less is equal to 4294967296 (which is `1 << 32`), then it's possible to just use indexes of type `uint32_t`.

```c
typedef uint32_t packed_ptr_t;

static void *IndexTranslate(packed_ptr_t ptr)
{
    return (void *) (REGION_START + ptr)
}

packed_ptr_t packed_ptr = Alloc(sizeof(int));
int *real_ptr = IndexTranslate(packed_ptr_t);
```

This works fine, however this way we lost the main advantage of C pointers -- type safety. For example with C pointers it is not possible to do the following


```c
void ConsumePointer(int *);

double *ptr;
ConsumePointer(ptr);
```

Any sane C compiler will reject this code with something like `error: incompatible pointer types passing 'char *' to parameter of type 'int *`. This is because it's unsafe to interpret one bytes of one type as byte of the other.

However, with indexes type information gets completely lost and following will compile and will cause bad things at runtime.

```c
typedef uint32_t packed_ptr_t;

static void *IndexTranslate(packed_ptr_t ptr)
{
    return (void *) (REGION_START + ptr)
}

void ConsumeInt(int *);

packed_ptr_t packed_ptr = Alloc(sizeof(char));

ConsumeInt(IndexTranslate(packed_ptr_t));
```

This is classic type confusion bug, which C is famous for. That bug causes tons of CVEs. Not good. I guess, I can skip the part where I explain how the code above may cause problems, so let's try to add a little bit of safety to indexes.

## Bringing type-safety to indexes

What if we could somehow attach the type to the index. Also it would be good if this tag will not consume any memory. Cool and modern languages have this feature.

For example in Rust it can be done as:

```rust
// Godbolt link https://godbolt.org/z/MKsf37a5b
use core::marker::PhantomData;
use core::mem::size_of;

#[derive(Default)]
struct Index<T>(u32, PhantomData<T>);

const fn foo() {
    assert!(size_of::<Index<usize>>() == size_of::<u32>());
}

// Poor man static assert
static TEST: () = foo();
static BASE: usize = 0x100000;

impl<T> Index<T> {
    fn as_ptr(&self) -> *const T {
        (BASE + self.0 as usize) as _
    }
}

```

In c++ it can be done as:

```cpp
// Godbolt link https://godbolt.org/z/zcnxYz9W7

#include <cstdint>

#define BASE    0x100000

template<class T>
struct Index {
    std::uint32_t idx;

    T *as_ptr() const {
        return reinterpret_cast<T *>(BASE + idx);
    }
};

static_assert(sizeof(Index<int>) == sizeof(std::uint32_t));

int *test(void)
{
    Index<int> idx;
    return idx.as_ptr();
}
```

But we are dealing with old-but-no-good C, so we can't afford generic programming without struggle.

### Template structs

As you all know, C has a preproccessor step. It is simple text substitution step that runs before compilation step. Using it, it's possible to create "template" structs in C like:

```c
#define TemplateStruct(T)   \
    struct Struct##T {      \
        T a, b, c;          \
    } test;

TemplateStruct(int) test = {};
```

With such macro it would be possible to construct a "generic" index, which will contain type. Moving on.

### Flexible arrays

Flexible arrays is kinda obscure C feature that allows to create DST (dynamically sized types) in C. However, it's completely unsafe and works not how you expect it to work.

Let's take a look at an example:

```c
// Godbolt link https://godbolt.org/z/aEPoEP6zG

struct Flex {
    int a;
    char arr[];
};
```

This declares a structure that contains one `int` and unknown number of `size_t` elements. Basically non-flex part of the struct can be viewed as header for the data in `arr`. In memory it's laid out as following (assuming `sizeof(int)` is 4):

```
|x| - is one byte of payload

  Flex
   v
   |x|x|x|x|x|x|x|...
   ^       ^
   a      arr
  
```

And `sizeof(struct Flex)` is equal to `sizeof(int)`. This looks like a solution to the problem -- flexible array is a "type tag" that costs no memory. That's all?

No. Let's take a look at this struct.

```c
// Godbolt link https://godbolt.org/z/aEPoEP6zG

struct FlexLong {
    int a;
    unsigned long arr[];
};
```

Should be `sizeof(int)`, shouldn't it? It should, but we are in C. `unsigned long` has bigger alignment requirements (8 bytes on LP64 systems) than `int`, so struct's size is padded to be multiply of it. As a result `sizeof(struct FlexLong)` is 8. This time memory layout will look like:

```
|x| - is one byte of payload

  Flex
   v
   |x|x|x|x| | | | |x|x|x|...
   ^       ^       ^
   a      padd    arr
  
```

Eh... Looked promising. Maybe there are some language extensions that can help?

### Attributes

Long time ago people noticed that writing code in ISO C is pain in the ass. In case you didn't, take a look at beautiful paper [How ISO C became unusable for operating systems development](https://arxiv.org/pdf/2201.07845). TL;DR nobody is writing ISO C now, everybody uses some superset of C that is more usable in the real life.

For example Linux kernel won't ever build with optimization level less that `-01`, since it  relies on a compiler optimizations. Also it heavily replies on language extensions provided by mainstream compilers. [GCC](https://gcc.gnu.org/onlinedocs/gcc-12.2.0/gcc/C-Extensions.html) and [Clang](https://clang.llvm.org/docs/LanguageExtensions.html) have a wide support for various extensions that make life in C easier.

I won't go into the details about all extensions, since there are a lot of them. I will focus only on the one we need: `__attribute__((packed))`. The attribute can be attached to structure type and it forces a compiler to remove all paddings (if you don't know what padding is, read this [SO answer](https://stackoverflow.com/questions/4306186/structure-padding-and-packing]). For example:

```c
struct Test {
    int a;
    char b;
};

_Static_assert(sizeof(struct Test) == sizeof(int) * 2);

struct TestPacked {
    int a;
    char b;
} __attribute__((packed));

_Static_assert(sizeof(struct TestPacked) == sizeof(int) + sizeof(char));
```

This way it's possible to use flexible array hack from previous part as type tag without size increase:

```c
// Godbolt link https://godbolt.org/z/xMP9Ec8Tv

struct FlexLongPacked {
    int a;
    unsigned long arr[];
} __attribute__((packed));

_Static_assert(sizeof(struct FlexLongPacked) == sizeof(int));
_Static_assert(_Alignof(struct FlexLongPacked) == 1);
```

This works, but `packed` removes all alignment requirements from the struct. This may sound OK, but in reality it is not. We are not in the 90s, so I won't tell you that this will cause crashes, no. On any modern hw unaligned access won't trap, however it **may** cause longer access. HW is faster when loads and stores work with aligned memory.

To fix it, it's possible to use another extension `__attribute__((aligned(_Alignof(type)))`
```c
// Godbolt link https://godbolt.org/z/a77ParErY

struct FlexLongPacked {
    int a;
    unsigned long arr[];
} __attribute__((packed, aligned(_Alignof(int))));

_Static_assert(sizeof(struct FlexLongPacked) == sizeof(int));
_Static_assert(_Alignof(struct FlexLongPacked) == _Alignof(int));
```

This way struct will be naturally aligned and will have zero-cost type tag.

### Constructing type-safe packed pointer

Now we have all needed tools to construct a type-safe packed pointer type. Let's try

```c
// Godbolt link https://godbolt.org/z/Kf8PY9WMz
typedef uint32_t packed_ptr_t;

#define PackedPtr(type)     \
struct {                    \
    packed_ptr_t ptr;       \
    type __dont_touch[];    \
} __attribute__((packed, aligned(_Alignof(packed_ptr_t))))

void test(void)
{
    PackedPtr(int) var;

    _Static_assert(sizeof(var) == sizeof(packed_ptr_t));
}
```

It works! Let's try to put this pointer in a struct.

```c
// Godbolt link https://godbolt.org/z/r7W8fPh6E
typedef uint32_t packed_ptr_t;

#define PackedPtr(type)     \
struct {                    \
    packed_ptr_t ptr;       \
    type __dont_touch[];    \
} __attribute__((packed, aligned(_Alignof(packed_ptr_t))))

struct Test {
    PackedPtr(int) ptr;
    int a;
};
```

GCC stays silent, however clang is more pedantic in such case:

```
warning: field 'ptr' with variable sized type 'struct (unnamed struct at <source>:12:5)' not at the end of a struct or class is a GNU extension [-Wgnu-variable-sized-type-not-at-end]
   12 |     PackedPtr(int) ptr;
```

Fair complain. Compiler does not know that this `__dont_touch` is a tag, but not a flexible array. In latter case it's definitely an error and should be reported. It's possible to use ugly pragmas to suppress this warning, but I hate it. Let's look for another solution.

After loud swearing and couple of hours I came up with.

```c
// Godbolt link https://godbolt.org/z/hE9nGzobc

#include <stdint.h>

typedef uint32_t packed_ptr_t;

#define PackedPtr(type)          \
struct {                         \
    struct {                     \
        packed_ptr_t ptr;        \
        union {                  \
            type __dont_touch;   \
        } __dont_touch[];        \
    } __attribute__((packed)) __inner[1];                \
} __attribute__((aligned(_Alignof(packed_ptr_t))))

struct Test {
    PackedPtr(int) ptr;
    int a;
    PackedPtr(int) ptr1;
};

void test(void)
{
    PackedPtr(unsigned long) var;

    _Static_assert(sizeof(var) == sizeof(packed_ptr_t), "");
    _Static_assert(_Alignof(typeof(var)) == _Alignof(packed_ptr_t), "");
}
```

Don't ask me why and how, it just works. Now let's add helper macros for this type:

```c
// Godbolt link https://godbolt.org/z/P4E7Wxhnj

#include <stdint.h>

#define BASE    0x100000

typedef uint32_t packed_ptr_t;

#define PackedPtr(type)          \
struct {                         \
    struct {                     \
        packed_ptr_t ptr;        \
        union {                  \
            type __dont_touch;   \
        } __dont_touch[];        \
    } __inner[1];                \
} __attribute__((packed, aligned(_Alignof(packed_ptr_t))))

#define PackedPtrGetPointerType(kPtr) typeof(&(kPtr).__inner[0].__dont_touch[0].__dont_touch)

#define PackedPtrGetPointer(kPtr)   \
    (PackedPtrGetPointerType(kPtr))((uintptr_t)(kPtr).__inner[0].ptr + BASE)

#define ASSERT_TYPES_COMPATIBLE(type1, type2)   _Static_assert(__builtin_types_compatible_p(type1, type2), "")

#define PackedPtrAssign(kPtr, pointer)  do {                                           \
    ASSERT_TYPES_COMPATIBLE(PackedPtrGetPointerType(kPtr), typeof(pointer));           \
    /* Here you likely want to check that pointer is in bounds */                      \
    (kPtr).__inner[0].ptr = BASE - (uintptr_t)(pointer);                               \
} while (0);

void test(int *someRealPtr, void *otherPtr)
{
    PackedPtr(int) ptr;

    PackedPtrAssign(ptr, someRealPtr);

    int *real = PackedPtrGetPointer(ptr);

    // Warns
    unsigned *realWrong = PackedPtrGetPointer(ptr);
    // Does not compiles
    PackedPtrAssign(ptr, otherPtr);
}
```

`PackedPtrGetPointerType(kPtr)` returns inner pointer type (i.e. `type *`). `PackedPtrGetPointer(kPtr)` return strongly typed pointer to the underlying data. `PackedPtrAssign` constructs packed pointer from a C pointer.

`PackedPtrAssign` does some magic by verifying that passed pointer is compatible with tag type. Otherwise type safety invariant is not maintained, since the original pointer type may not match the resulting type.

That's all? I'd hope so, but we need one small step further. Consider following:

```c
void test(void)
{
    PackedPtr(int[10]) ptr;
}
```

It definitely won't compile, because it will be expanded to

```c
struct {
    struct {
        packed_ptr_t ptr;
        union {
            int[10] __dont_touch;
        } __dont_touch[];
    };
}
```

, which is not valid C. We need our tag to be a pointer rather than a plain type. This way it will be possible to store any type tags.

First thing that comes to mind is adding `*` after type like

```diff
#define PackedPtr(type)           \
struct {                          \
    struct {                      \
        packed_ptr_t ptr;         \
        union {                   \
-            type __dont_touch;   \
+            type *__dont_touch;  \
        } __dont_touch[];         \
    } __inner[1];                 \
} __attribute__((packed, aligned(_Alignof(packed_ptr_t))))
```

But `int[10]*` is also not a valid type. To get the type of the pointer to some value we can use following trick

```c
#define PackedPtrConstructTypeTag(type)     typeof(&(type){})
```

Inside `typeof` operator we contruct literal of type `type` and take pointer to it. Something like [declval](https://en.cppreference.com/w/cpp/utility/declval.html) for C.

Using this macro we now can finish our type-safe zero-cost packed pointer in C. Not in ISO C, but in real C.

## Full code

Full code can be found in the next section and on my [github](https://github.com/pskrgag/packed_ptr). Macros here are intended to be used as building blocks, since they expect external base from user. You can build you macros on top of it.

```c
/*
 * Copyright (c) 2025-2025 Pavel Skripkin
 *
 * Permission to use, copy, modify, and distribute this software for any
 * purpose with or without fee is hereby granted, provided that the above
 * copyright notice and this permission notice appear in all copies.
 *
 * THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES
 * WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF
 * MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR
 * ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES
 * WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN
 * ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF
 * OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
 */

#ifndef PACKED_PTR_H
#define PACKED_PTR_H

/**
 * Packed pointer type that hold offset from some base address and zero-sized type tag.
 */
#define PackedPtr(inner_type, packed_ptr_t)                           \
	struct {                                                      \
		struct {                                              \
			packed_ptr_t ptr;                             \
			union {                                       \
				PackedPtrConstructTypeTag(inner_type) \
					__dont_touch;                 \
			} __dont_touch[];                             \
		} __attribute__((packed)) __inner[1];                 \
	} __attribute__((aligned(_Alignof(packed_ptr_t))))

/**
 * Generates type tag. Internal use only.
 */
#define PackedPtrConstructTypeTag(type) typeof(&(type){})

/**
 * Retrieves the type of the underlying pointer.
 *
 * @param[in] kPtr     packed pointer lvalue
 *
 * @return             type of underlying pointer
 */
#define PackedPtrGetPointerType(kPtr) \
	typeof((kPtr).__inner[0].__dont_touch[0].__dont_touch)

/**
 * Retrieves the C pointer from packed pointer.
 *
 * @param[in] kPtr     packed pointer lvalue
 * @param[in] base     base address of the allocator memory range
 *
 * @return             C pointer to the original allocation
 */
#define PackedPtrGetPointer(kPtr, base)                                    \
	(PackedPtrGetPointerType(kPtr))((uintptr_t)(kPtr).__inner[0].ptr + \
					(base))

/**
 * Asserts that two types are equal.
 *
 * @param[in] type1    first type
 * @param[in] type2    second type
 *
 * @return             nothing, or fires compiler error in case of type mismatch
 */
#define ASSERT_TYPES_COMPATIBLE(type1, type2) \
	_Static_assert(__builtin_types_compatible_p(type1, type2), "")

/**
 * Assigns value to the packed pointer.
 *
 * @note Fires compiler error in case of `pointer` type does not match type of the packed pointer
 *
 * @param[in] kPtr     packed pointer lvalue
 * @param[in] pointer  pointer to memory allocation
 * @param[in] base     base address of the allocator memory range
 *
 * @return             nothing, or fires compiler error in case of type mismatch
 */
#define PackedPtrAssign(kPtr, pointer, base)                           \
	do {                                                           \
		ASSERT_TYPES_COMPATIBLE(PackedPtrGetPointerType(kPtr), \
					typeof(pointer));              \
		(kPtr).__inner[0].ptr = (base) - (uintptr_t)(pointer); \
	} while (0);

#endif /* PACKED_PTR_H */
```
