---
layout: post
title:  "Implementing C++ coroutines with Apple dispatch queues"
tags: c++ macos ios xcode
---

<!-- Links -->

[objc-helpers]: https://github.com/gershnik/objc-helpers
[co-dispatch-doc]: https://github.com/gershnik/objc-helpers/blob/master/doc/CoDispatch.md
[coobjc]: https://github.com/alibaba/coobjc
[cppref-coro]: https://en.cppreference.com/w/cpp/language/coroutines
[chen-coro]: https://devblogs.microsoft.com/oldnewthing/20210504-01/?p=105178
[outcome]: https://ned14.github.io/outcome/
[isptr]: https://github.com/gershnik/intrusive_shared_ptr
[for-await]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for-await...of

<!-- End Links -->

**TL;DR**: If you are here looking to use C++ coroutines with Apple dispatch queues you can find a single header library I ended up writing [here][objc-helpers]. Documentation is [here][co-dispatch-doc]. You get the whole shebang: awaiting asynchronous code, "promisifying" callbacks, awaiting other coroutines and asynchronous generators all working on dispatch queues accessible from C++, ObjectiveC++ or a mixture of both. This post is mostly about my experiences creating this library; various issues and observations I had while creating it.

<hr />
<p></p>
Some time ago I needed to write some fairly sequential code on macOS that had to do many things asynchronously: running code on dispatch queues directly via `dispatch_async`, reacting to callbacks (also running on various queues under the hood) and dispatching more code back and so on. A pretty typical stuff that ends up being messy and hard to read when written in the traditional way. At some point I thought to myself: why can't I write this using C++ coroutines I've heard so much about? They've been around for a while now so there probably is a library that can do this already. Surely in 2023 we can make asynchronous C++ code look as neat as JavaScript async/await one!

In pursuit of this goal, I embarked on an internet search and, to my surprise, came back almost empty handed. Perhaps my google-foo and github-foo are severely lacking and there is a library somewhere that already does all I needed but I couldn't find anything like that. The obvious thing all searches bring is [coobjc][coobjc] but this is not a C++ coroutine library. It implements coroutines from scratch in assembly for use in plain ObjectiveC/Swift. While nice, it is quite large and is a library you have to link with. Surely none of this is needed in C++ with native coroutine support? I also found a few C++ libraries that implement generic coroutines running on an abstract "Executor" concept which, presumably, I could implement using dispatch queues. Some of them were quite old (using `std::experimental` namespace for coroutines) and I wasn't sure if they are still usable. They are also relatively large. More generally it felt that spending time learning how to customize some library and fighting its quirks for something that ought to be very simple to begin with is not a time well spent.

So at the end I decided to simply try to write my own and see how easy or hard it is. If I get stuck or this snowballs beyond all proportion I could always go back to one of these libraries and try to implement an "Executor" instead.

With this decision made, the next problem I encountered was a surprising lack of resources on how to implement asynchronous coroutines. There is [cppreference][cppref-coro] of course but it is a reference, not a guide and its examples mainly show synchronous coroutines. It is of course possible to "figure it all out" from just the reference but it's a lot of work that somebody surely already done. 

Many other coroutine guides I found on the web turned out to be nothing more that gentler restatements of what cppreference says. 

What I found to be pure gold in terms of content is Raymond Chen's [long series of articles on the topic of coroutines][chen-coro]. I follow his blog but ignored this series when it came out because I had zero interest in the topic at the time. My only complaint about it is that it follows a "didactic" approach where it spoon-feeds you small pieces of information in each installment building upon previous "lessons". You have to follow along diligently without skipping steps even if their topics seem at first far removed from what you are actually trying to accomplish! 

However, if you persevere, you will be rewarded with great insight on how to make asynchronous coroutines work including how to make them lock-free and avoiding many subtle pitfalls.

Armed with the knowledge I got from this series I was able to quickly implement four major parts of coroutines on dispatch queues:
1. Ability to await any callable dispatched on a queue like this:
  ```cpp
    int i = co_await co_dispatch(some_queue, []() {
        return 7;
    });
    ```
2. Ability to "promisify" a call with asynchronous callback to turn it into something that can be co-awaited.
  ```cpp
    int i = co_await makeAwaitable(some_queue, [](auto promise) {
        someAppleApiWithCallback(^(int result, NSError * err){
            if (err)
                promise.failure(std::runtime_error(...));
            promise.success(result);
        });
    });
    ```
3. Ability for one coroutine to await another:
  ```cpp
    DispatchTask<int> someCoroutine() {
        //do asynchronous things
        co_return 7;
    }
    DispatchTask<int> anotherCoroutine() {
        int i = co_await someCoroutine();
    }
    ```
4. Last, but not least an ability to simply switch execution queues 
  ```cpp
    co_await resumeOn(some_other_queue);
    ```

The first three cost exactly 1 memory allocation (for "promise" object) and are lock-free in transferring return value back to awaiter. Note that typical `dsispatch_async(queue, ^{})` call also costs one memory allocation to copy the block.

The last one performs no memory allocations at all.

One challenge I had implementing it was to create a "return value carrier" type that can transfer any possible return value: void, pointer, reference (rvalue or lvalue) or an object (not necessarily default constructible) from the asynchronous call to the awaiter. You'd think such a type would be a part of standard library but as far as I know it isn't. There are bits and pieces like `std::reference_wrapper` (that bizarrely doesn't work with rvalue references) but nothing that can do the whole thing. Also note that the "value carrier" needs to support carrying over a possible exception for non-noexcept code. [Outcome][outcome] library provides such type and as far as I know there is a proposal to add something like this to the standard (and quite a few knock-offs on the web) but I didn't want to have an external dependency.

I ended up rolling out my own based on `std::variant` which ended up working quite well because I could also introduce "not initialized" state which is useful to catch bugs and would come in handy for generators.

The second challenge I had was how to communicate information about _which queue to resume the awaiter on_. By default an awaiter would be resumed on the same queue the asynchronous code was executing upon. This works great in some scenarios but in other you want the resumption happen on a different queue (often the main UI one). With `co_dispatch` I could of course pass this info as an argument but what about awaiting another coroutine call? I needed something generic that would work with everything in the same way.

Unlike `operator new` for which you can define custom arguments there is no such facility for `operator co_await`. Coroutines allow another route where the promise type can "peek at" coroutine arguments (it seems this was a hack made to support allocators). So in principle I could define something like "if a coroutine first (or last) parameter is of type ResumeOn then that's where it will be resume" but doing this seems bizarre and hacky.

At the end I decided to simply use a type of builder pattern on an "awaitable" returned from async calls before it is being awaited. Thus you have to say:

```cpp
co_await co_dispatch(queue, []() { ... }).resumeOn(someQueue);
...

DispatchTask<> someTask();

co_await someTask().resumeOn(someQueue)

```

Which at the end looks kind of neat even though it wouldn't be my first choice of how to do it.

Yet another problem was how to deal with absence or presence of exceptions. The code could be optimized quite a bit _if_ it knew that the asynchronous operation does not throw exceptions. With `co_dispatch` this is easy - just look whether the callable argument is `noexcept`. But with general coroutines and `makeAwaitable` this is impossible. With general coroutines specifying `noexcept` _doesn't mean what you'd suppose it does_. It only says that it produces the task object without exceptions. The normal body of the coroutine can throw as much as it wants to. There appears to be no way to mark a coroutine as "not throwing at all" so I ended up asking user to optionally specify it in return type:

```cpp
DispatchTask<int, SupportsExceptions::No> someCoro() {
    ....
}
```

and similarly for `makeAwaitable`. A bit ugly but it works.


The last challenge had to do with making `makeAwaitable`. Most Apple callbacks are blocks and blocks capture their variables _always by copy_ so move-only objects don't work with them. This meant that the "promise" passed to a black needs to be copyable. The original Raymond Chen's design was to only have `std::unique_ptr`s to promises and move them around. Very nice and safe but I couldn't use it here. So I had to use reference counting and, since I didn't want to pay 2-pointer price of `std::shared_ptr` I needed intrusive reference counting. Which meant adding a small intrusive smart pointer class to the code since I didn't want to bring in an external library (like [this one][isptr]). Not a big deal but annoying and yet another missing feature of C++ standard library.

With all these features implemented I turned to implementing asynchronous generators. As fas as I can tell this was not covered in any of Raymond Chen's posts so I had to figure it out on my own. Fortunately, it turned out to be not too complicated. Once you have implemented an asynchronous coroutine the generator is not too different. The main difference is that the generator can be restarted and that you need to distinguish between "completion" via `co_yield` and `co_return` somehow. Adding restarting to the lock-free state machine turned out to be fairly simple. Distinguishing between the completion modes came for free because the `std::variant` that I used to hold the result already had an "uninitialized" value (indicated via `std::mosostate`). Thus, I could use this mode upon completion to indicate `co_return` completion.

The real issue came in defining the interface to _use_ the generator. The obvious choice is to use some form of "iterator" but C++ has no defined "awaitable iterator" concept. Neither does it posses some form of for [for await][for-await] loop. In C++23 there is `std::generator` but it only works with _synchronous_ coroutines and so doesn't need to be awaited and can expose traditional STL iterator interface. In absence of any clear precedent I defined the following iteration pattern:

```c++
DispatchGenerator<int> generate() {
    co_yield 1;
    co_yield 2;
    ...
}

for (auto it = co_await generate().begin(some_queue); it; co_await it.next()) {
    int value = *it;
}
```

The obvious difference from traditional iterators is that you need to `co_await` calls to `begin()` and `next()`. 

The second difference is that I use a named method: `next()` rather than `++` to advance the iterator. This is because `++it` canonically returns a reference to the iterator. Not that anybody ever uses it, but that what it is supposed to do. The "increment" in this case returns an awaitable.

The third difference that you check for an end by just converting iterator to bool rather than comparing it to some "sentinel". The whole sentinel thing is a hack to make naturally "single iterator" entities conform to "iterator-pair" protocol of STL. There is nothing to conform to here so I could as well make the typing shorter.

With this protocol you are only allowed to call `bool(it)` and `*it` when `co_await` have returned. Doing it at other times produces UB. 

Additionally `*it` can only be called once for each element. For efficiency it will always try to _move_ the underlying value so if the value has non-trivial "moved from" state calling it a second time will produce that.

Any exceptions thrown from the generator will be reported from awaiting `begin()` or `next()` calls as would be expected from a "normal" iterator.

All in all, this seems to be a fairly recognizable "for-each-iterator" loop and I am pretty happy with it. Of course language support for some form of `for co_await(...)` would have been better. Obviously you cannot use any STL algorithms with iterators that need to be awaited either. 

Once this was done I pretty much had all the pieces of a comprehensive library in place. To make it fully comprehensive I also provided convenience wrappers for Dispatch I/O methods (after all one of the obvious uses for coroutines is asynchronous I/O!). With the building blocks above it is as simple as this:

```c++
struct DispatchIOResult {
  DispatchIOResult(dispatch_data_t data, int error) noexcept;
  ...
};

inline auto co_dispatch_read(dispatch_fd_t fd, 
                             size_t length, 
                             dispatch_queue_t queue) {
  return makeAwaitable<DispatchIOResult, SupportsExceptions::No>(
    [fd, length, queue](auto promise) {
      dispatch_read(fd, length, queue, ^ (dispatch_data_t data, int error) {
        promise.success(data, error);
    });
  });
}
```

Beyond this there is some additional complexity in supporting "delayed resumption" (via `dispatch_after`) and mixing C++ and ObjectiveC++ code in the same executable. You can find the details in the [usage doc][co-dispatch-doc] if interested.

I might do a separate post on the last mixing languages issue some day since it is its own large-ish topic with significance to other projects as well.

All in all doing this exercise was relatively straightforward (big thanks to Raymond Chen!) but with lots of unnecessary friction dues to deficiencies of C++ standard library and what feels like gaps in coroutine machinery. Hope you find it useful!


