---
title: Annotations for Reflection
tag: constexpr
---

Annotations for Reflection
===

**Document number**: P3394R0 [\[Latest\]](https://wg21.link/p3394) [\[Status\]](https://wg21.link/p3394/status)
**Date**: 2024-09-14
**Authors**:
    - Wyatt Childers (wcc@edg.com)
    - Dan Katz (dkatz85@bloomberg.net)
    - Barry Revzin (barry.revzin@gmail.com)
    - Daveed Vandevoorde (daveed@edg.com)
**Audience**: Evolution



## Introduction

Ever since writing [P1240R2](https://wg21.link/p1240r2) ("Scalable Reflection in C++")], but more so since [P2996R5](https://wg21.link/p2996r5) ("Reflection for C++26"), we have been requested to add a capability to annotate declarations in a way that reflection can observe. For example, Jeremy Ong presented compelling arguments in a post to the SG7 reflector: https://lists.isocpp.org/sg7/2023/10/0450.php. Corentin Jabot also noticed the need while P1240 was evolving and wrote [P1887R0](https://wg21.link/p1887) ("Reflection on Attributes"), which proposes syntax not entirely unlike what we present here.


In early versions of P2996 (and P1240 before that), a workaround was to encode properties in the template arguments of alias template specializations:

```cpp
template <typename T, auto... Annotations>
using Noted = T;

struct C {
    Noted<int, 1> a;
    Noted<int*, some, thing> b;
};
```

It was expected that something like `type_of(^C::a)` would produce a reflection of `Noted<int, 1>` and that can be taken apart with metafunctions like `template_arguments_of` — which both preserves the type as desired (`a` is still an `int`) and allows reflection queries to get at the desired annotations (the `1`, `some`, and `thing` in this case).

There are problems with this approach, unfortunately:

* It doesn't work for all contexts where we might want to annotate declarations (e.g., enumerators).
* It doesn't directly express the intent.
* It turns out that providing access to aliases used in the declaration of reflected entities raises difficult questions and P2996 is therefore likely going to drop the ability to access that information (with the intention of resurrecting the capability with a later paper once specification and implementation challenges have been addressed).

In this paper, we propose simple mechanisms that more directly support the ability to annotate C++ constructs.

## Motivating Examples

We'll start with a few motivating examples for the feature. We'll describe the details of the feature in the subsequent section.

These examples are inspired from libraries in other programming languages that provide some mechanism to annotate declarations.

### Command-Line Argument Parsing

Rust's [clap](https://docs.rs/clap/latest/clap/) library provides a way to add annotations to declarations to help drive how the parser is declared. We can now [do the same](https://godbolt.org/z/1sdqdEfro):

```cpp
struct Args {
    [[=clap::Help("Name of the person to greet")]]
    [[=clap::Short, =clap::Long]]
    std::string name;

    [[=clap::Help("Number of times to greet")]]
    [[=clap::Short, =clap::Long]]
    int count = 1;
};


int main(int argc, char** argv) {
    Args args = clap::parse<Args>(argc, argv);

    for (int i = 0; i < args.count; ++i) {
        std::cout << "Hello " << args.name << '\n';
    }
}
```

Here, we provide three types (`Short`, `Long`, and `Help`) which help define how these member variables are intended to be used on the command-line. This is implemented on top of [Lyra](https://github.com/bfgroup/Lyra).

When run:

```
$ demo -h
USAGE:
  demo [-?|-h|--help] [-n|--name <name>] [-c|--count <count>]

Display usage information.

OPTIONS, ARGUMENTS:
  -?, -h, --help
  -n, --name <name>       Name of the person to greet
  -c, --count <count>     Number of times to greet

$ demo -n wg21 --count 3
Hello wg21
Hello wg21
Hello wg21
```

While `Short` and `Long` can take explicit values, by default they use the first letter and whole name of the member that they annotate.

The core of the implementation is that `parse<Args>` loops over all the non-static data members of `Args`, then finds all the `clap`-related annotations and invokes them:

```cpp
template <typename Args>
auto parse(int argc, char** argv) -> Args {
    Args args;
    auto cli = lyra::cli();

    // ...

    template for (constexpr info M : nonstatic_data_members_of(^^Args)) {
        auto id = std::string id(identifier_of(mem));
        auto opt = lyra::opt(args.[:M:], id);

        template for (constexpr info A : annotations_of(M)) {
            if constexpr (parent_of(type_of(A)) == ^^clap) {
                // for those annotions that are in the clap namespace
                // invoke them on our option
                extract<[:type_of(A):]>(A).apply_annotation(opt, id);
            }
        }

        cli.add_argument(opt);
    }
};
```

So, for instance, `Short` would be implemented like this:

```cpp
namespace clap {
    struct ShortArg {
        // optional isn't structural yet but let's pretend
        optional<char> value;

        constexpr auto operator()(char c) const -> ShortArg {
            return {.value=c};
        };

        auto apply_annotation(lyra::opt& opt, std::string_view id) const -> void {
            char first = value.value_or(id[0]);
            opt[std::string("-") + first];
        }
    };

    inline constexpr auto Short = ShortArg();
}
```

Overall, a fairly concise implementation for an extremely user-friendly approach to command-line argument parsing.

### Test Parametrization

The pytest framework comes with a decorator to [parametrize](https://docs.pytest.org/en/7.1.x/how-to/parametrize.html) test functions. We can now do [the same](https://godbolt.org/z/vnT51585G):

```cpp
namespace N {
    [[=parametrize({
        Tuple{1, 1, 2},
        Tuple{1, 2, 3}
        })]]
    void test_sum(int x, int y, int z) {
        std::println("Called test_sum(x={}, y={}, z={})", x, y, z);
    }

    struct Fixture {
        Fixture() {
            std::println("setup fixture");
        }

        ~Fixture() {
            std::println("teardown fixture");
        }

        [[=parametrize({Tuple{1}, Tuple{2}})]]
        void test_one(int x) {
            std::println("test one({})", x);
        }

        void test_two() {
            std::println("test two");
        }
    };
}

int main() {
    invoke_all<^^N>();
}
```

When run, this prints:

```
Called test_sum(x=1, y=1, z=2)
Called test_sum(x=1, y=2, z=3)
setup fixture
test one(1)
teardown fixture
setup fixture
test one(2)
teardown fixture
setup fixture
test two
teardown fixture
```

Here, `parametrize` returns a value that is some specialization of `Parametrize` (which is basically an array of tuples, except that `std::tuple` isn't structural so the implementation rolls its own).

The rest of the implementation looks for all the free functions named `test_*` or nonstatic member functions of class types that start with `test_*` and invokes them once, or with each parameter, depending on the presence of the annotation. That looks like this:

```cpp
consteval auto parametrization_of(std::meta::info M) -> std::meta::info {
    for (auto a : annotations_of(M)) {
        auto t = type_of(a);
        if (has_template_arguments(t) and template_of(t) == ^^Parametrize) {
            return a;
        }
    }
    return std::meta::info();
}

template <std::meta::info M, class F>
void invoke_single_test(F f) {
    constexpr auto A = parametrization_of(M);

    if constexpr (A != std::meta::info()) {
        // if we are parametrized, pull out that value
        // and for each tuple, invoke the function
        // this is basically calling std::apply on an array of tuples
        constexpr auto Params = extract<[:type_of(A):]>(A);
        for (auto P : Params) {
            P.apply(f);
        }
    } else {
        f();
    }
}

template <std::meta::info Namespace>
void invoke_all() {
    template for (constexpr std::meta::info M : members_of(Namespace)) {
        if constexpr (is_function(M) and identifier_of(M).starts_with("test_")) {
            invoke_single_test<M>([:M:]);
        } else if constexpr (is_type(M)) {
            template for (constexpr std::meta::info F : nonstatic_member_functions_of(M)) {
                if constexpr (identifier_of(F).starts_with("test_")) {
                    invoke_single_test<F>([&](auto... args){
                        typename [:M:] fixture;
                        fixture.[:F:](args...);
                    });
                }
            };
        }
    };
}
```

## Proposal

The core idea is that an *annotation* is a compile-time value that can be associated with a construct to which attributes can appertain. Annotation and attributes are somewhat related ideas, and we therefore propose a syntax for the former that builds on the existing syntax for the latter.

At its simplest:

```cpp
struct C {
    [[=1]] int a;
};
```

Syntactically, an annotation is an attribute of the form `= expr` where `expr` is a _`constant-expression`_ (which syntactically excludes, e.g., _`comma-expression`_) to which the glvalue-to-prvalue conversion has been applied if the expression wasn't a prvalue to start with.

Currently, we require that an annotation has structural type because we're going to return annotations through `std::meta::info`, and currently all reflection values must be structural.

### Why not Attributes?

Attributes are very close in spirit to annotations.  So it made sense to piggy-back on the attribute syntax to add annotations. Existing attributes are designed with fairly open grammar and they can be ignored by implementations, which makes it difficult to connect them to user code. Given a declarations like:

```cpp
[[nodiscard, gnu::always_inline]]
[[deprecated("don't use me")]]
void f();
```

What could reflecting on `f` return? Because attributes are ignorable, an implementation might simply ignore them. Additionally, there is no particular value associated with any of these attributes that would be sensible to return. We're limited to returning either a sequence of strings. Or, with [P3294](https://wg21.link/p3294), token sequences.

But it turns out to be quite helpful to preserve the actual values without requiring libraries to do additional parsing work. Thus, we need to distinguish annotations (whose values we need to preserve and return back to the user) from attributes (whose values we do not). Thus, we looked for a sigil introducing a general expression.

Originally, the plus sign (`+`) was considered (as in P1887), but it is not ideal because a prefix `+` has a meaning for some expressions and not for others, and that would not carry over to the attribute notation.  A prefix `=` was found to be reasonably meaningful in the sense that the annotation "equals" the value on the right, while also being syntactically unambiguous. We also discussed using the reflection operator (`^`) as an introducer (which is attractive because the annotation ultimately comes back to the programmer as a reflection value), but that raised questions about an annotation itself being a reflection value (which is not entirely improbable).

As such, this paper proposes annotations as distinct from attributes, introduced with a prefix `=`.

### Library Queries

We propose the following set of library functions to work with annotations:

```cpp
namespace std::meta {
  consteval bool is_annotation(info);

  consteval vector<info> annotations_of(info item);            // (1)
  consteval vector<info> annotations_of(info item, info type); // (2)
  template<class T>
    consteval optional<T> annotations_of(info item);           // (3)

  consteval info annotate(info item,
                          info value,
                          source_location loc = source_location::current());
}
```

`is_annotation` checks whether a particular reflection represents an annotation.

We provide three overloads of `annotations_of` to retrieve all the annotations on a particular item:

1. Returns all the annotations.
2. Returns all the annotations `a` such that `type_of(a) == type`.
3. Returns the annotation `a` such that `dealias(type_of(a)) == ^^T`. If no such annotation exists, returns `nullopt`. If more than one such annotation exists and all the values are not template-argument-equivalent, this call is not a constant expression.

And lastly, `annotate` provides the ability to programmatically add an annotation to a declaration.

### Additional Syntactic Constraints

Annotations can be repeated:

```cpp
[[=42, =42]] int x;
static_assert(annotations_of(^x).size() == 2);
```

Annotations spread over multiple declarations of the same entity accumulate:

```cpp
[[=42]] int f();
[[=24]] int f();
static_assert(annotations_of(^f).size() == 2);
```

Annotations follow appertainance rules like attributes, but shall not appear in the *attribute-specifier-seq* of a *type-specifier-seq* or an *empty-declaration*:

```cpp
struct [[=0]] S {};  // Okay: Appertains to S.
[[=42]] int f();     // Okay: Appertains to f().
int f[[=0]] f();     // Ditto.
int [[=24]] f();     // Error: Cannot appertain to int.
[[=123]];            // Error: No applicable construct.
```

To avoid confusion, annotations are not permitted after an *attribute-using-prefix*. So this is an error:

```cpp
[[using clang: amdgpu_waves_per_eu, =nick("weapon")]]
int select_footgun(int);
```
Instead, use:

```cpp
[[using clang: amdgpu_waves_per_eu]] [[=nick("weapon")]]
int select_footgun(int);
```

### Implementation Experience

The core language feature and the basic query functions have been implemented in the EDG front end and in Dan Katz's P2996 Clang fork (with option `-freflection-latest`), both available on Compiler Explorer.