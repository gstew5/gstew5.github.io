---
layout: post
title: Implementation-defined Behavior in C
tags: C semantics
published: true
---

I've become interested recently in how we should think about C programs that perform implementation-defined operations, in particular, conversions between nonnull pointers and integers. 

Mostly, this is just idle curiosity on my part. 

On the other hand, there are some interesting issues surrounding implementation-defined behavior, portability, and formal models of compiler correctness. I don't go deep into these issues here---this post is mostly just a high-level overview, with some unanswered questions.

First, definitions: 

### Implementation-defined Behavior

The C11 standard defines "implementation-defined behavior" as 
> unspecified behavior where each implementation documents how the choice is made.
> (3.4.1, [C11 draft standard](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf))

Here are some examples:

* pointer-to-integer (and integer-to-pointer) conversions, whenever the pointer/integer is nonnull/nonzero
* representation of signed integers (two's complement, one's complement, etc.)
* the extent to which the compiler respects the inline function specifier
* ... and many more (J.3)

### Unspecified Behavior 

"Unspecified behavior" (3.4.4), in turn, is behavior that either has two or more possible outcomes, or involves the use of an "unspecified value," which is any valid value of the relevant type (int, float, etc.). 

All of the following are unspecified:

* the order in which subexpressions are evaluated (and the order in which any resulting side-effects occur) in function calls, evaluation of `&&`, `||`, `?:`, and comma operators
* the order in which `#` and `##` operations are evaluated during preprocessing
* ... and many more (J.1)

### Integer-Pointer Conversion

The most interesting kind of implementation-defined behavior, from my perspective, is conversion between (void, nonnull) pointers and integers. These kinds of conversions can be legitimately useful in some situations (consider tagged pointers in GCs). On the other hand, they can also quickly degenerate.

To illustrate how things can get messy, consider the following C program. I believe this program is well-defined according to the C standard, but please correct me if I'm wrong!

{% raw %}
<pre>
int g(int* x) { return ((uintptr_t)(void*)x <= 0xbffff980); }

static int f(void) {
  int a = 0;
  return g(&a);
}

int main(void) {  return f(); }
</pre>
{% endraw %}

What integer does this program return? 

If we do a mental trace, we pretty quickly get to the key bit:

{% raw %}
<pre>
return ((uintptr_t)(void*)x <= 0xbffff980);
</pre>
{% endraw %}

The first cast to `void*` is always well-defined.
The conversion `(uintptr_t)x`, on the other hand, is only implementation defined, i.e., "[...] has unspecified behavior, where each implementation documents how the choice is made." 
(`uintptr_t` is an unsigned integer type guaranteed not to screw up the integer-pointer conversion.)

Let's assume for a moment that we're trying to determine the result without reference to any particular compiler (C verification and bugfinding tools do this all the time).

The result of the conversion must just be unspecified, right? We don't have an implementation to "document how the choice is made."

Which means that `(uintptr_t)x` is an "unspecified value" of unsigned integer type (that is, any unsigned integer representable as a `uintptr_t`). All we can say, then, about the less-than-or-equal-to comparison is that it is either 0 or 1, but we don't know which. The program is nondeterministic.

Another way to think about this is: If we want to treat the code as truly portable, and, e.g., verify its result once and for all for __all__ implementations, we need to consider all possible implementations of implementation-defined operations such as pointer-to-integer casts.

### gcc

Things get a bit weirder if we start thinking about the behavior of the program above with reference to a particular C implementation, e.g., gcc, clang, or CompCert.

gcc, for example, documents pointer-to-integer casts as follows:

> A cast from pointer to integer discards most-significant bits if the pointer representation is larger than the integer type, sign-extends(2) if the pointer representation is smaller than the integer type, otherwise the bits are unchanged. 

If we assume gcc, we should assume that pointer-integer casts have this behavior, which should at least resolve the nondeterminism we saw earlier, right? (Because we're fixing a particular implementation.)

We can experiment by compiling the program, running it, and checking its exit code. 

When compiled without optimizations, I get exit code 1 on my machine.

```  
> gcc -O0 f.c
> ./a.out
> echo $?
> 1
```

When compiled with optimizations turned on, I get exit code 0.

``` 
> gcc -O3 f.c
> ./a.out
> echo $?
> 0
```

What's going on?

In the second compilation, gcc inlines function `f` at `f`'s callsite in `main`. Inlining, in turn, changes the absolute position on the stack at which local variable `a` is allocated, which causes the less-than-or-equal comparison to switch from `1` to `0`. 
(To see this behavior on your own platform, you'll probably have to choose the integer `0xbffff980` a bit craftily, and turn address-space randomization off.)

Here we've uncovered a second source of unspecified, or nondeterministic, behavior in the program above. Although the cast is now guaranteed to produce a well-defined result, there's no reason we should assume that `a` is allocated at any particular location in memory. The C standard guarantees that the address of an object remains constant only during the _lifetime_ of the object, not across program executions (6.2.4, footnote 33).

### CompCert

While I was at it, I thought I'd compile the program above with [CompCert](http://compcert.inria.fr), just to see what happens.
CompCert, if you haven't seen it before, is a verified compiler for a large subset of C, written and proved correct in the Coq theorem prover. The CompCert distribution includes both the compiler (with correctness proof) and an executable C interpreter.

Compiling the program above with CompCert results in the same behavior (different return values, depending on whether `f` is inlined or not). To get CompCert to inline `f`, I had to explicitly add the `inline` function specifier. To get different return values, I also had to pick a slightly different integer constant.

More interesting is what happens when you "run" the program above using CompCert's C interpreter `ccomp -interp`:

```
Stuck state: in function g, expression <ptr> <= -1073743488
Stuck subexpression: <ptr> <= -1073743488
ERROR: Undefined behavior
```

The `ERROR: Undefined behavior` indicates that the CompCert C semantics got stuck when attempting to execute the pointer-integer cast `(uintptr_t)x` in `g`.

If you read the CompCert C semantics, you see that the cast to `uintptr_t` leaves the pointer a pointer (it's classified as a "neutral" cast by CompCert). The comparison, between the pointer and integer, then gets stuck.

The question I'm left with: Is getting stuck at this point a valid "unspecified behavior"?

