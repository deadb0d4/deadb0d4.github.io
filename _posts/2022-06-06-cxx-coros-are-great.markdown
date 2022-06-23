---
layout: post
title:  "[wip] C++ coroutines are great"
date:   2022-06-06 16:21:21 +0300
---

# Introduction

In this post, I will be outlining my learning of c++20 coroutines. When I was reading about it, I thought the feature is weird and is probably a subject for replacement. As I learned more and more about it, I gradually changed my mind. Now, I firmly believe the c++20 coroutines are great and will be improving over time.

For recreational purposes, I write algorithms in c++ every so often. While one often doesn’t need coroutines for writing CPU-bound code, I thought it would be fun to use coroutines for the following toy problem.

# The problem

We have the following 'node' class for representing a binary tree.
```c++
struct Node {
  int value;
  Node* left;
  Node* right;
};

bool IsLeaf(const Node& p) {
  return !p.left && !p.right;
}
```
I call a node _a leaf_ if both its `left` and `right` pointers are null. Given a binary tree root, return a sum of values in its leaves.

# The benchmark

To test different solutions, I used full binary trees. One can generate such trees like that:

```c++
/// @brief: a full binary tree with `n` leaves
/// so `n` is assumed to be 2^k
Node* MakeFullTree(int n, int s = 0) {
  auto p = new Node(s);
  if (n == 1) {
    return p;
  }
  n /= 2;
  p->left = MakeFullTree(n, s + 1);
  p->right = MakeFullTree(n, s + 2 * n);
  return p;
}
```
I parametrise the benchmarks below by the number of leaves in a full binary tree.

# Basic solution

Without coroutines, one can use the following generic code.
```c++
void Dfs(Node* p, auto&& leaf_callback) {
  if (!p) {
    return;
  }
  if (IsLeaf(*p)) {
    leaf_callback(p->value);
    return;
  }
  Dfs(p->left, leaf_callback);
  Dfs(p->right, leaf_callback);
}

int res = 0;
Dfs(root, [&res](int x) { res += x; });
```
We use the depth-first search to traverse the tree and a callback to sum up values. Compared to what follows, it's ridiculously fast:

```
----------------------------------------------------------------
Benchmark                      Time             CPU   Iterations
----------------------------------------------------------------
FastDfsBench/16384         40150 ns        40067 ns        12763
FastDfsBench/1048576     3243753 ns      3241907 ns          215
FastDfsBench/4194304    13239332 ns     13225222 ns           54
FastDfsBench/16777216   55297925 ns     55155500 ns           10
```

Wouldn't it be nice to iterate over leaf values though? This question got me into brushing-up my knowledge of coroutines. I mean something like this:
```c++
int res = 0;
for (int leaf_value : LeafValues(root)) {
  res += leaf_value;
}
```

# Iterators

One can create a class for traversing a tree. It should store the traversal state so far and have methods `begin` and `end` for the range-based for loop above. Effectively, it means to rewrite recursion with for-loops and a custom stack.

Let's solve our problem using a coroutine-based generator instead.

# Stackful coroutines

Stackful coroutines provide an intuitive way of writing asynchronous code.

One can build user-level coroutines using the [Boost Coroutine2][boost-coro] library. Solving the problem at hand, we can do something like this:

```c++
// Fun fact: the generic code above could be applied there
auto DfsImpl(Node* p, coro_t::push_type& yield) {
  if (!p) {
    return;
  }
  if (IsLeaf(*p)) {
    // Jump to the caller returning the next value
    yield(p->value);
    return;
  }
  DfsImpl(p->left, yield);
  DfsImpl(p->right, yield);
}

auto LeafValues(Node* p) {
  // create a channel-like handle; this is effectively the entry point
  // to the coroutine execution flow: call to coro will resume the coroutine
  coro_t::pull_type coro([p](coro_t::push_type& yield) {
    DfsImpl(p, yield);
  });
  return coro;
}
```

If you think about the program stack you will notice something tricky is going on above. Here is the snippet from the library documentation.

>
> If coroutines were called exactly like routines, the stack would grow with every call and would never be popped. A jump into the middle of a coroutine would not be possible, because the return address would be on top of stack entries.
>
> The solution is that each coroutine has its own stack and control-block (fcontext_t from Boost.Context). Before the coroutine gets suspended, the non-volatile registers (including stack and instruction/program pointer) of the currently active coroutine are stored in the coroutine's control-block. The registers of the newly activated coroutine must be restored from its associated control-block before it is resumed.
>

Since such coroutines have their own stack space, the word 'stackful' applies.

The benchmark for this solution gives a good upper bound for any coroutine-based solution.
```
-----------------------------------------------------------------
Benchmark                       Time             CPU   Iterations
-----------------------------------------------------------------
FiberDfsBench/16384        766208 ns       765749 ns          776
FiberDfsBench/1048576    49170744 ns     49153214 ns           14
FiberDfsBench/4194304   197069094 ns    197005250 ns            4
FiberDfsBench/16777216  801769458 ns    801597000 ns            1
```

Let's now try c++20 coroutines.

# C++ coroutines: an intro

I don’t feel comfortable writing yet another c++20 coroutines introduction. That is because there are already a bunch of excellent resources. I like [1,3-5,7](#references) below.

For what follows, I assume we agree on the points below.
- To _suspend_ is to stop the current execution for a while and execute something else. A _suspension_ might happen only inside a _coroutine function_. A _coroutine function_ is a function with a specific return type that has keywords like `co_{await,yield,return}` inside its body

- Every time a _coroutine function_ is created, a _memory region_ for its locals and state of execution will be allocated. We will informally call this memory region as well as the code inside the coroutine function the _coroutine frame_

- We use `co_await` keyword to say to the compiler that we _may_ suspend here. The `co_yield` keyword is actually `co_await promise.yield_value`. The `co_return` keyword is a bit more involved, but does `co_await` in the end

# Generator

A basic example of a coroutine is a generator. It's typically used as follows:
```c++
generator SimpleYields(int x, int y) {
  co_yield x;
  co_yield y;
  co_yield x + y;
}
// usage
for (int e : SimpleYields(1, 3)) {
  cout << e << ' '; // 1 3 4
}
```
The [cppcoro generator](https://github.com/lewissbaker/cppcoro/blob/master/include/cppcoro/generator.hpp) is a good reference implementation.

If we have the `generator` class above we can write something like this:
```c++
generator LeafValues(Node* p) {
  if (!p) {
    co_return;
  }
  if (IsLeaf(*p)) {
    co_yield p->value;
    co_return;
  }
  for (int x : LeafValues(p->left)) {
    co_yield x;  // value propagates to the caller
  }
  for (int x : LeafValues(p->right)) {
    co_yield x;  // value propagates to the caller
  }
}
```
Both memory and time complexities are bad though. Since `co_yield`s are nested, each value gets propagated `O(log(n))` times. Hence, the time complexity is `O(n log(n))` instead of `O(n)` for our benchmark. Also, each recursive call of `LeafValues` will heap-allocate the coroutine frame because there is no way to predict its size on the caller site.

Let's ignore the memory for a moment. We can always rely on modern recycling allocators if anything. The time complexity is more interesting to reduce now.

# Symmetric transfer

Let's say we are in a coroutine frame `A` and we want to 'open' a new coroutine frame `B`. To mitigate nested yields, we want to transfer control such that
- `A` suspends and we continue at `B`
- If `B` suspends we go back to `A`'s caller (not to `A`)

This is what the symmetric transfer feature does in particular. In code, it looks as follows.
```c++
coroutine_handle<> await_suspend(coroutine_handle<> a_handle) const noexcept {
  // return `B`'s handle to transfer suspend A
  // and 'fully' transfer control to B
  return b_handle;
}
```
The method `await_suspend` is a required method of an awaitable object. Hence, we need to 'await' nested calls. One can overload operator `co_await` for this:
```c++
auto Coro::operator co_await() {
  // return an awaitable object with await_suspend above
  // I skip its code for brevity
  return Wait{handle};
}
```
Now, we can draft a solution to our problem:
```c++
Coro CoroDfs(Node* p) {
  if (!p) {
    co_return;
  }
  if (IsLeaf(*p)) {
    // return to the original caller (not to the upper frame)
    co_yield p->value;
    co_return;
  }
  co_await CoroDfs(p->left);  // transfer control inside
  co_await CoroDfs(p->right); // transfer control inside
}
```
Post-factum, this code makes perfect sense. The logic stays the same as in the example. We just annotated the points where suspension (will) may happen for the compiler.

At this point, we've covered the first yield. When it happens, we will return to the caller side, but we can't go back to the coroutine just yet. Nested coroutine handles (frames) naturally form a call stack. In one way or another, we should be able to interact with this stack.

Some people store a pointer to the previous frame inside each coroutine handle. This way, it’s effectively an intrusive list data structure.

Instead, I decided to allocate the stack itself<sup>[ a](#notes)</sup> so that I could use its space to avoid a bunch of heap allocations, and I also could access coroutine frames directly.

To use a custom allocator, we define class-level overloads.
```c++
static void* Coro::promise_type::operator new(size_t count) {
  // here `task` is a static, one should use thread_local
  // but thread locals are slow on macOS, while OK on Linux
  return task->stack.New(count);
}

static void Coro::promise_type::operator delete(void* ptr) {
  // no need to delete here
}
```
We got a fun issue though. The heap allocation elision optimization (HALO) may happen even if there is custom `new` and `delete`. And it did happen with my Release build. I used the following to stop HALO.
```c++
// force compiler to not inline this method
__attribute__((noinline)) Coro Coro::promise_type::get_return_object() {
  return {std::coroutine_handle<promise_type>::from_promise(*this)};
}
```
I am not sure if it guarantees to prevent HALO every time, though.

Now, we can go back to the coroutine from which we yielded like that:
```c++
void Task::Resume() {
  coroutine_handle<Coro::promise_type>::from_address(
      stack.BackAddress()).resume();
}
```

The caller's code looks like this in my benchmark.
```c++
for (auto call = CoroDfs(root); task->HasStack(); task->Resume()) {
  // check that some value is yield'ed
  if (task->value.IsSet()) {
    res += task->value.Pop();
  }
}
```

The benchmark for this code:
```
-----------------------------------------------------------
Benchmark                 Time             CPU   Iterations
-----------------------------------------------------------
BM_Coro/16384        280575 ns       279798 ns         2446
BM_Coro/1048576    18082648 ns     18012395 ns           38
BM_Coro/4194304    71140171 ns     71137889 ns            9
BM_Coro/16777216  316724208 ns    315827000 ns            2
```

It isn't as fast as the first DFS with a callback, but it's still better than the naive stackful code above. You can find the full benchmark [here][my-code].

# Wrap-up

Initially, I had two issues with c++20 coroutines.

First of all, it seemed you couldn’t control heap allocations. Now, I would say you can but it's hard (as always). I came across [this proposal](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p2477r0.html) on control over heap allocation elision. Ironically, it might be more important to block HALO rather than to ensure it.

Lastly, there are about a dozen of customization points. It means one has to write a lot of boilerplate code and understand it before doing something interesting. A good point here is that c++20 coroutines are _a low-level assembly-language for coroutines_<sup>[3](#references)</sup> by design.

I see a small inconvenience with these customization points, though. Libraries will probably present their own coroutines and awaitables for their concurrency model. Hence, its users won't be able to just say `co_await` without integration with the library. It's ok but it might be cumbersome to do.

For now, c++ coroutines are superior already. One can control virtually anything that is happening inside their coroutines.

# References

1. [C++ Coroutines Do Not Spark Joy][cpp-do-not-spark-joy] by Malte Skarupke
2. [Boost Coroutine2 Page][boost-coro]
3. [Asymmetric Transfer](https://lewissbaker.github.io/) by Lewis Baker
4. [C++20: Building a Thread-Pool With Coroutines](https://blog.eiler.eu/posts/20210512/) by Michael Eiler
5. [My tutorial and take on C++20 coroutines](https://www.scs.stanford.edu/~dm/blog/c++-coroutines.html) by David Mazières
6. [Full source code as a benchmark][my-code]
7. [Coroutines in LLVM](https://llvm.org/docs/Coroutines.html)

[cpp-do-not-spark-joy]: https://probablydance.com/2021/10/31/c-coroutines-do-not-spark-joy/
[boost-coro]: https://www.boost.org/doc/libs/1_72_0/libs/coroutine2/doc/html/index.html
[coro-cppref]: https://en.cppreference.com/w/cpp/language/coroutines
[my-code]: https://github.com/deadb0d4/incidental-cxx/blob/develop/benchmarks/coro_dfs.cpp#L157

# Notes

<sub>a. I didn't want to use [a custom stack](#iterators) but ended up having it? Not really. The user of `Coro` doesn't have to write a custom stack every time there is a recursion. Instead, I use a generic call stack which I hide from users </sub>
