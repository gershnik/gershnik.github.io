---
layout: page
exclude: true
title: IOCP and Overlapped I/O
---

{::options parse_block_html="true" /}

<div style="background-color:#f55c51;border-radius:10px;padding:10px">
<sm>This is an archive copy of an old FAQ for microsoft.public.win32.programmer.networks newsgroup I once wrote. People still follow links to it so I keep it alive. Note that it was written in 2005-2006 and information here may be obsolete.

Use at your own risk</sm>
</div>
<br>

* TOC
{:toc}
{: #toc}

[claa]: https://learn.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-getqueuedcompletionstatus
[clac]: https://learn.microsoft.com/en-us/windows/win32/api/ioapiset/nf-ioapiset-postqueuedcompletionstatus

---

## Sample code
{: #sample}

**Problem**

Where can I find sample code that demonstrates how to use IOCP?

**Solution**

* Platform SDK has a sample in `samples\netds\winsock\iocp` directory
* Felix Kasza provided [sockhim](https://web.archive.org/web/20150324082152/http://win32.mvps.org/network/sockhim.html) sample
* If you use MFC (<u>which I definitely do not recommend</u>) there is [this article](https://web.archive.org/web/20100115084913/http://chonga.pe.kr/computer/programming/ms/windows_iocp.html) by Ruediger Asche

[Back to top](#toc)

---

## Signaling all threads
{: #signal_all}

**Problem**

How do I signal all threads in a thread pool associated with IOCP? Usually this problem arises when you want to cleanly stop all threads or otherwise let them perform some special operation.

**Solution**

In general the whole idea of thread pools is that threads are indistinguishable from each other and it is up to IOCP to decide who is going to be invoked for each request.

If all you need is to gracefully terminate all the threads just post the special completion status exactly N times where N is the number of threads in the pool. Since each thread will die after receiving the message and not call [GetQueuedCompletionStatus()][claa] again you are guaranteed to kill them all.

If you _really_ need to terminate a specific thread you can do this. Create a custom `OVERLLAPED`-derived structure that includes a thread ID of the destination thread. Post a pointer to this structure with [PostQueuedCompletionStatus()][clac]. The thread that gets the message should check the thread ID. If it is its own the thread should terminate. Otherwise it should call [PostQueuedCompletionStatus()][clac] again with the same parameters and **wait until the destination thread dies**. This could be very slow but at least you are guaranteed that eventually the destination thread will get the message.

[Back to top](#toc)

---

## What is faster?
{: #speed}

**Problem**

What is "faster" IOCP, overlapped I/O, event notifications or blocking calls? What I/O technology should I use to make my software faster?

**Solution**

If raw speed is all that you need then simple blocking I/O calls is the fastest method available. However, you rarely need raw speed by itself. For most servers, for example, _scalability_, that is an ability to support huge amounts of clients without exponentially bigger delays, is probably more important than raw speed. In such situations IOCP has an inherent advantage, because, it allows you to utilize your CPUs much more effectively. For GUI client application, on the other hand, _responsiveness_, that is being able to react to user commands without perceptible delays, is the key concept. For them, event notifications or overlapped I/O will allow you to structure your code in such a way as to allow immediate interaction with UI.

Bottom line is: make sure you understand the exact requirements of your application (beyond "make it fast"), learn the pluses and minuses of available technologies and select the one that best fits your needs. There is no silver bullet here.

[Back to top](#toc)

---

## Thread count
{: #thread_count}

**Problem**

How many threads do I need in my thread pool? I have heard that I should have one worker thread per CPU. Is this true?

**Solution**

No. What you need is to have one work**ing** thread per CPU under a full load. If all your working thread does is to perform some calculations then, yes, one such thread per CPU will be ideal. In practice, however, most worker threads will eventually block waiting for some I/O. This is more common than people expect and often not under your control. An innocent call to OS may use some I/O internally or you may call some DLL that does that inside. In such situations you need other available worker threads to keep the CPU busy. How many? The only way to answer this question is to monitor application behavior under realistic conditions. Even better, adjust the number at runtime based on application behavior. One further point to keep in mind is that if you suddenly discover that you need a lot of worker threads per CPU you may be doing something wrong. Having lots of threads (even if most of them are sleeping) compromises system performance because of context switching and other bookkeeping overhead. In such situation it may be better to split your thread pool in two (i.e. use to completion ports). The first pool will handle network requests while the second do whatever background processing that keeps threads sleeping. The two pools can communicate using the standard producer-consumer pattern.

[Back to top](#toc)