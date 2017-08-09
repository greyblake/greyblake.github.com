---
layout: post
title: "Porting a Rust library to C"
date: 2017-08-10 10:16
comments: true
draft: true
categories: rust C FFI
---

* [Intro](/blog/2017/08/10/porting-rust-library-to-c/#intro)
* [Hello from Rust example](/blog/2017/08/10/porting-rust-library-to-c/#hello-from-rust-example)
* [Naming conventions](/blog/2017/08/10/porting-rust-library-to-c/#naming-conventions)
* [Representing plain enums](/blog/2017/08/10/porting-rust-library-to-c/#plain-enums)
* [Strings](/blog/2017/08/10/porting-rust-library-to-c/#strings)
  * [Passing a string to a function](/blog/2017/08/10/porting-rust-library-to-c/#passing-a-string-to-a-function)
  * [Returning a string from a function](/blog/2017/08/10/porting-rust-library-to-c/#returning-a-string-from-a-function)
* [Dealing with structures](/blog/2017/08/10/porting-rust-library-to-c/#structs)
  * [Representing a structure](/blog/2017/08/10/porting-rust-library-to-c/#representing-a-structure)
  * [Returning a structure](/blog/2017/08/10/porting-rust-library-to-c/#returning-a-structure)
* [Complex enums?](/blog/2017/08/10/porting-rust-library-to-c/#complex-enums)
* [Conclusion](/blog/2017/08/10/porting-rust-library-to-c/#conclusion)
* [Links](/blog/2017/08/10/porting-rust-library-to-c/#links)

## <a name="intro"></a> Intro

Recently I've ported [whatlang](https://github.com/greyblake/whatlang-rs) library to C ([whatlang-ffi](https://github.com/greyblake/whatlang-ffi))
and I'd like to share some experience.

DISCLAIMER: I am not a professional C/C++ developer, so it means:

* I will describe some things that may look very obvious.
* The outcome probably will not be a 100% idiomatic C code.
* If you know how some things can be done better, please let me know by writing a comment.


## <a name="hello-from-rust-example"></a> Hello from Rust example

First let's make a minimal C program, that calls Rust.

```
cargo new whatlang-ffi
cd whatlang-ffi
mkdir examples
```

Add this to `Cargo.toml`:

    [lib]
    name = "whatlang"
    crate-type = ["staticlib", "cdylib"]

It tells cargo that we want to compile a static library and get `.so` object.


In `src/lib.rs` we implement a small function that prints a message to stdout:

```rust
#[no_mangle]
pub extern fn print_hello_from_rust() {
    println!("Hello from Rust");
}
```

<!--more-->

To explain `#[no_mangle]` and `extern` let me extract some quotes from:
[FFI with Haskell and Rust](https://mgattozzi.com/haskell-rust) article:


{% blockquote %}
The #[no_mangle] tells the Rust compiler not to do anything weird with the symbols of this function when compiled because we need to be able to call it from other languages.
This is needed if you plan on doing any FFI. Not doing so means you won't be able to reference it in other languages
{% endblockquote %}

{% blockquote %}
extern means this is externally available outside our library and tells the compiler to follow the C calling convention when compiling
{% endblockquote %}

Let's compile our lib:

    cargo build --release

It creates `target/release/libwhatlang.so` file.

Now using `nm` tool we can check that `libwhatlang.so` really contains `print_hello_from_rust()` function:

    nm -D ./target/release/libwhatlang.so  | grep hello
    0000000000003190 T print_hello_from_rust

Then we need `src/whatlang.h` header file with a function declaration:

```c
void print_hello_from_rust();
```

And finally a C program itself (we put it into `examples/hello.c`):

```c
#include "whatlang.h"

int main (void) {
    print_hello_from_rust();
}
```

Let's compile!

    gcc -o ./examples/hello ./examples/hello.c -Isrc  -L. -l:target/release/libwhatlang.so

This produces `examples/hello` binary, which we can run:

    ./examples/hello
    Hello from Rust

During the development process we'll likely need to recompile and run the program frequently.
To automate this let's create a `Makefile` with few commands:

```
GCC_BIN ?= $(shell which gcc)
CARGO_BIN ?= $(shell which cargo)

run: clean build
	./examples/hello
clean:
	$(CARGO_BIN) clean
	rm -f ./examples/hello
build:
	$(CARGO_BIN) build --release
	$(GCC_BIN) -o ./examples/hello ./examples/hello.c -Isrc  -L. -l:target/release/libwhatlang.so
```

Now, we can run `make run` to recompile `lib.rs`, `hello.c` and run `hello` binary.

In the rest for the article I'll go through common problems, design decisions and pitfalls I faced.


## <a name="naming-conventions"></a>Naming conventions

Since C does not have namespaces (some people may [disagree](https://stackoverflow.com/questions/4396140/why-doesnt-ansi-c-have-namespaces))
I had to stick to some rules in order to avoid name collision with other libraries and confusion:

* Every function, type or constant starts with `<library>_` prefix. Examples: `whatlang_detect()`, `whatlang_lang`.
* If a function is associated with a particular format then its name has format  `<library>_<type>_<name>`. Example: `whatlang_lang_code()`.

Similar logic rules apply to everything else. It may seem too verbose,
but as I see it's a pretty common approach for many C libraries.

## <a name="plain-enums"></a> Representing plain enums

In whatlang I have enum [Lang](https://docs.rs/whatlang/0.4.1/whatlang/enum.Lang.html), which
is probably is main entity in the library. First  I had to add [`#[repr(C)]`](https://doc.rust-lang.org/nomicon/other-reprs.html#reprc)
to make rust compiler represent the data in memory in the same way as C does:

```rust

#[repr(C)]
pub enum Lang {
    Aka,
    Amh,
    Arb,
    // and so on
}
```

`Lang` represents 83 different languages, which can be encoded with 1 byte.
That was my initial assumption an it seemed to be correct, until later I uncovered some bugs.

I decided to convert Lang enum into `u8` with [std::mem::transmute](https://doc.rust-lang.org/std/mem/fn.transmute.html) function,
in order to figure out how it's encoded:

```
let lang_int: u8 = unsafe { std::mem::transmute(Lang::Eng) };
println!("lang_int = {:?}", lang_int);
```

and got the following error:

    = note: source type: whatlang::Lang (32 bits)
    = note: target type: u8 (8 bits)

Wow! So, actually the enum takes 4 bytes, instead of 1.
Replacing `u8` with `u32` makes things work as expected:

    lang_int = 14

Now it make sense, because English is on 15th position in the Lang declaration
(remember, counting starts with 0).

So in C such enum can be mapped to `uint32_t` type from `stdint.h`.
To define all the language I ended up with such list of constants:

```c
#include <stdint.h>

static const uint32_t WHATLANG_LANG_AKA = 0;
static const uint32_t WHATLANG_LANG_AMH = 1;
static const uint32_t WHATLANG_LANG_ARB = 2;
// and so on
```

It's quite verbose, so one would rather use scripting a language to generate such boilerplate code.

UPDATE: later I figured out, that actually without `#[repr(C)]` Rust optimizes memory and
uses 1 byte for `Lang` enum. So `uint32_t` can be replaced with `uint8_t`. It should work
as far as number of enum variants does not exceed 256.


### Returning a structure from a function

## <a name="strings"></a> Strings

First I recommend you to read the docs for [std::ffi::CStr](https://doc.rust-lang.org/std/ffi/struct.CStr.html)
and [std::ffi::String](https://doc.rust-lang.org/std/ffi/struct.CString.html)
from the standard library. Those types exist to represent C strings in Rust.

### <a name="passing-a-string-to-a-function"></a> Passing a string to a function

From C side it's relative simple: just pass a pointer to a string, like it's done here with argument `text`:

```
uint8_t whatlang_detect(char* text, struct whatlang_info* info);
```

On Rust side you'll need to convert a pointer into `&str` or `String` so you can manipulate the data as a string.

```rust
use std::ffi::CStr;
use std::os::raw::c_char;

pub extern fn whatlang_detect(ptr: *const c_char, info: &mut Info) -> u8 {
    let cstr = unsafe { CStr::from_ptr(ptr) };

    match cstr.to_str() {
        Ok(s) => {
          // Here `s` is regular `&str` and we can work with it
        }
        Err(_) => {
          // handle the error
        }
    }
}
```

In the example above the function accepts a raw pointer `*const c_char` (it can be also `*mut c_char` if you need to mutate data).
Then we transform it into `CStr` calling unsafe method `CStr::from_ptr(ptr)`.
Finally we're calling `CStr::to_str(&self)` function, which converts C string into `&str`.
This operation may fail, if the C string does not contain a valid UTF-8 sequence.

### <a name="returning-a-string-from-a-function"></a> Returning a string from a function

`Lang` provides some methods that return static strings, like `eng_name()` to get language name in English.
Example:

```
assert_eq!(Eng::Rus.eng_name(), "Russian")
```

My first thought was "I just can return a raw pointer to the string", so the initial solution was like:


C function declaration:
```c
char* whatlang_lang_eng_name(uint32_t lang);
```

Rust implementation:
```rust
#[no_mangle]
pub extern fn whatlang_lang_eng_name(lang: Lang) -> *const u8 {
    lang.eng_name().as_ptr()
}
```

But there is problem. C expects strings to be terminated with `\0` character, while  Rust actually
organizes static strings in a different way.
When I expected the output to be simple `Russian`, the output was the entire massive of static data:

    RussianRundiRomanianPortuguesePolishPersianPunjabiOromoOriya....

So, I've decided that I actually need to copy string from Rust static memory and ensure that
`\0` is added.


So I came up with the following function:

```c
void whatlang_lang_eng_name(uint32_t lang, char* buffer);
```

Now user needs to pass a pointer to a buffer, where result must be written.

Rust implementation:

```rust
extern crate libc;

#[no_mangle]
pub extern fn whatlang_lang_code(lang: Lang, buffer_ptr: *mut c_char) {
    // Here unwrap is safe, because whatlang always returns a valid &str
    let s = CString::new(lang.code()).unwrap();
    unsafe {
        libc::strcpy(buffer_ptr, s.as_ptr());
    }
}
```

First, we convert `&str` into `CString`. Then we use `libc::strcpy` from [libc](https://doc.rust-lang.org/1.6.0/libc/index.html)
crate to copy the string.

NOTE: it's responsibility of a caller to ensure, that the buffer size is big enough (at least 30 bytes).

## <a name="structs"></a> Dealing with structures

### <a name="representing-a-structure"></a> Representing a structure

I have the following flat Rust structure `Info`

```rust
#[repr(C)]
pub struct Info {
    lang: Lang,
    script: Script,
    confidence: f64
}
```

(where `Lang` and `Script` are plain enums), and it easily maps to `whatlang_info`:

```c
struct whatlang_info {
  uint32_t lang;
  uint32_t script;
  double confidence;
};
```

It could be slightly more complex with nested structures, but the idea stays the same.

### <a name="returning-a-structure"></a> Returning a structure

I guess it can be done at least in few different approaches.
The way I do it: a function receives a pointer to a preallocated memory for a structure as one of the arguments.

You've already seen `whatlang_detect` function above:

```
uint8_t whatlang_detect(char* text, struct whatlang_info* info);
```

Here `info` is pointer, where result must be written in case of success (`0` is returned).


Another way to do this is to return a pointer directly:

```c
struct whatlang_info* whatlang_get_info();
```

In this case Rust function must return boxed structure:

```rust
#[no_mangle]
pub extern fn whatlang_get_info() -> Box<Info> {
    let info = Info { lang: Lang::Ukr, script: Script::Cyrillic, confidence: 0.9 };
    Box::new(info)
}
```

NOTE: In this approach the memory for the structure is allocated by Rust, but it's responsibility of
C program to free it.

## <a name="complex-enums"></a> Complex enums?

There is also some thing, that I am actually not aware how do properly:
**how to represent complex Rust enum in C?**

Therefore I also don't know how gracefully represent `Result<T,E>` and `Option<T>`
types.

Maybe, it's not actually necessary. For know as a workaround my function
```c
uint8_t whatlang_detect(char* text, struct whatlang_info* info);
```

returns `0` in case of `Some` and `1` in case of `None`. It writes
a result to preallocated memory by a given pointer `info`.


But I would appreciate if you share some other insights about this.

There are also some things, that are not covered in this article like tuples and arrays.
But you may get some ideas from this [article](http://pramode.in/2016/09/13/using-unsafe-tricks-in-rust/).

## <a name="conclusion"></a> Conclusion

It was shown how to create C bindings for a Rust library.
It may not be something, that you would do often, but having such option is always nice.
It means also, that Rust libraries may be ported to plenty other languages that has FFI support,
and this sounds really cool!

Thanks for reading. Below you'll find some useful links that helped me during this investigation.

## <a name="links"></a> Links
* [The Rust Book, Foreign Function Interface](https://doc.rust-lang.org/book/first-edition/ffi.html) - section in the first edition for The Rust book, about how to do FFI.
* [The Nomicon book](https://doc.rust-lang.org/nomicon/) - entire book dedicated to unsafe programming in Rust.
* [LibC](https://doc.rust-lang.org/libc/x86_64-unknown-linux-gnu/libc/index.html) - a crate, that allows to call C function from Rust. You'll find here C type definitions, constants and standard functions.
* [Rust FFI: Sending strings to the outside world](https://thefullsnack.com/en/string-ffi-rust.html) - this article explains how to expose Rust strings for NodeJS.
* [FFI with Haskell and Rust](https://mgattozzi.github.io/2016/10/01/haskell-rust.html) - yet another blog article about FFI
* [Using unsafe tricks to examine Rust data structure layout](http://pramode.in/2016/09/13/using-unsafe-tricks-in-rust/)
* [Complex types with Rust’s FFI](https://medium.com/jim-fleming/complex-types-with-rust-s-ffi-315d14619479)
* [netinfo-ffi](https://github.com/kaegi/netinfo-ffi) - was my initial source of inspiration, where I could find some examples.
* [https://github.com/greyblake/whatlang-ffi](whatlang-ffi) - whatlang C bindings described in this post
