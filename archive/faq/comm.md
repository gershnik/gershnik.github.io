---
layout: page
exclude: true
title: Windows Sockets
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

[aaac]: https://learn.microsoft.com/en-us/troubleshoot/windows-server/networking/reserve-a-range-of-ephemeral-ports
[aaag]: https://learn.microsoft.com/en-us/previous-versions/troubleshoot/windows/win32/data-segment-tcp-winsock
[baab]: http://www.iana.org
[waaa]: https://learn.microsoft.com/en-us/windows/win32/api/winsock2/nf-winsock2-bind
[waab]: https://learn.microsoft.com/en-us/windows/win32/api/winsock2/nf-winsock2-getsockname
[waac]: https://learn.microsoft.com/en-us/windows/win32/api/winsock2/nf-winsock2-connect
[iaab]: http://groups.google.com/group/microsoft.public.win32.programmer.networks/msg/126e038e2f29181c

---

## Server port
{: #server_port}

**Problem**

What port should I use for my server? How to pick an "unused" port? How to make my clients know what port my server is listening on?

**Solution**

The main thing to remember that a client sends the first message to the place where server is listening. In fact this is the only logical distinction between client and server in TCP/IP. After the first message the client and server are indistinguishable unless higher level protocol makes them so. Another thing to keep in mind is that there could be many intermediate stations that an IP packet has to pass before reaching the server. Each of these intermediaries mat decide to drop the packet if he doesn't like the destination for whatever reason (usually security).

With the above in mind the port that the server should listen to is mainly determined by client capabilities and the kind of network path you expect the traffic to traverse. There are few common approaches that people use to handle various clients and network environments. These are described below

**Use well-known ports**
<div style="margin-left:20px">
A well-known port is a port registered with [IANA][baab] for well-known protocols. For example for HTTP the pre-defined port is 80, while for FTP they are 20 and 21. Note that IANA distinguishes between well-known ports (range 0-1023) and reserved ports (range 1024-49151). However, for the purposes of this discussion the difference is not important. The full list of currently assigned port numbers can be found here. If you have enough resources you can probably register a port for your custom protocol with them. The advantage of using well-known ports is that normal protocol clients will support them by default so users (whether human or not) won't have to find out the port. Then various firewalls and other security devices are more likely to allow a well-known port than an unknown one. The disadvantage of a well-known port is that there is usually only one of them per-protocol. So if some other server had already taken your port you will have to use another one (see below). Still, when it is possible using a well-known port is the best solution. <span style="color:red">Note that what you should never do is to hijack a well-known port for an unrelated protocol</span>. This is sometimes tempting to let your traffic pass firewalls unnoticed but you should resist the temptation. First, firewalls are getting smarter and may not allow something that doesn't look like HTTP to pass on port 80. Second, your users won't appreciate the fact that installing your application along with a normal web server becomes impossible.
</div>

**Use an agreed upon port**
<div style="margin-left:20px">
In this method both client and server have to agree about what port to use in advance. The information about the port is passed by some method external to actual protocol. A trivial example is when somebody tells you that the URL to his site is `http://www.someserver.com:225`. This method have an advantage of being able to use any port. However, the disadvantage is that you need to somehow communicate the information about the port to the client. If your system doesn't allow for such communication you will have a problem. Another disadvantage is that firewalls and other security devices will in general not allow your traffic to come through. You users will have to reconfigure their firewalls to be able to use your server. And there is still a problem of possible collision with another server.
</div>

**Use random port**
<div style="margin-left:20px">
In this method server picks an unused random port upon startup. This can be accomplished by passing port 0 to [`bind()`][waaa] call. Of course, now you have the task of somehow letting the client know which port you are actually using. To discover the port you got from the `bind()` call see [this topic](#getsockname). After you have discovered the port you will need to record it in some well-know location where clients can find. A typical example would be storing the port in Active Directory or any other LDAP directory you may have. This method has an advantage of never suffering from port collisions but this comes at a price. You will need some infrastructure for the "well-known location" mentioned above. And there is still the problem with firewalls not passing traffic on unknown port numbers.
</div>

**Reuse some other server**
<div style="margin-left:20px">
To state it simply, consider making your server a web application, a SOAP web service, a COM or RPC server etc. In this way you will be insulated from low-level networking details like ports and from much of the worries about firewalls and proxies. This solution is often overlooked when designing network applications. The decision to develop your own server should be only taken as a last resort when the requirements of your application don't allow anything else. Writing fast, maintainable and secure server is an incredibly hard task and very few people can actually do it. If you are not an expert in this area chances are you will create a disaster. This could be a nice learning experience but not something to build commercial products upon.
</div>

[Back to top](#toc)

---

## Reserving a port
{: #reserve}

**Problem**

Can I reserve a port on a user's machine so that only my process will be able to use it?

**Solution**

No. The only way to reserve a specific port is to [`bind()`][waaa] a socket to it.

On Win2k and better there is a way to make sure that a program that doesn't do explicit `bind()` before [`connect()`][waac] will not be assigned a port in a given range. Details can be found [here][aaac].

Note that this doesn't reserve the port so it won't be accessible to other applications. All it does is to prevent inadvertent collisions with applications that rely on OS to assign their port numbers.

**Discussion**

Sockets follow the common paradigm that most OSes provide for their resources - "to use a resource is the only way to reserve it". The same holds for files, mutexes, pipes and pretty much everything else. Conceptually this is similar to a performance with tickets that do not reserve seat numbers (like in US movie theaters). The only way to ensure you will get a seat you want is to get there first. When the resource is plentiful and there is not much competition, the "first come first serve" is the simplest and cheapest way to implement resource allocation. This is probably the reason it is so popular among API designers. However, just like with movies, things may get ugly if there is not enough seats for everyone or when two persons desire the same seat. In such cases the normal solution is to assign seat numbers to tickets. People, then can sort out who sits were in advance without fighting and congestion at "runtime" i.e. when performance is going on. Such system is of course harder to implement. Conceptually, something like this could be done in software too, and I believe it has been done on occasion. Going back to TCP/IP ports we know that unlike files or mutexes, they are a limited resource with some, like 80, much more popular than others. This is why we almost never hear that somebody wants to reserve a filename but people often want to reserve ports. Because of this it may make sense for an OS to introduce some port-reservation mechanism for applications that can be used separately from communication code. Imagine how nice it would be if when a user configures your server and select a port 80 you could show him a nice dialog box saying "Port 80 is already reserved by Apache Web Server application. Please select another one". If anyone from MS is reading this please consider it a modest request for a feature 🙂

[Back to top](#toc)

---

## Local address and port
{: #getsockname}

**Problem**

How to find out the local IP address and/or port my socket is using?

**Solution**

This question usually arises on client which usually doesn't explicitly [`bind()`][waaa] its sockets. However it may also occur on server that uses random port technique. The solution is pretty simple. The address and port that are used by a bound socket can always be found by calling [`getsockname()`][waab]. Note that while this function will always return the local port the IP address will not always be known. To quote MSDN:

> The getsockname function does not always return information about the host address when the socket has been bound to an unspecified address, unless the socket has been connected with connect or accept (for example, using ADDR_ANY). A Windows Sockets application must not assume that the address will be specified unless the socket is connected. The address that will be used for the socket is unknown unless the socket is connected when used in a multihomed host. If the socket is using a connectionless protocol, the address may not be available until I/O occurs on the socket.

[Back to top](#toc)

---

## 200ms delay
{: #200ms}

**Problem**

I've got 200ms delay before the server answers.

**Solution**

Your protocol or usage of sockets is wrong. Consider redesigning your application

**Explanation**

The following excellent explanation is due to Alun Jones and appeared in this [newsgroup post][iaab].

> Whenever you hear "200 ms delay", you should automatically start thinking "I wonder if this is an interaction between my software, the Nagle algorithm, and delayed ACK?"
>
> [...]
>
> Here's the way it all works:
> 
> Think of it as if the Nagle algorithm affects senders, and the delayed ACK algorithm affects the receiver.
>
> The Nagle algorithm aims to cut down on short TCP segments, by collecting them all together when the network is busy handling your previous segments. It does this by sending only if one of the following is true:
>
> 1. All previous data has been acknowledged.
> 2. There is more than a full segment's worth of data to send.
>
> The delayed ACK algorithm says that ACKs should be sent only under one of the following situations:
>
> 1. We are sending other data that we can "piggyback" onto.
> 2. We have received two segments of data.
> 3. 200ms has elapsed since the first piece of unacknowledged data was received.
>
> The classic example of Nagle and delayed ACK interaction is of a sender issuing two small send()s, and then waiting for a recv() of data that is a response to the second send().  As you can see from checking the above, the first send() goes immediately onto the network, because there is no unacknowledged data preceding it (item 1 of the Nagle algorithm's list). The data makes its way to the receiver, who runs down his list, and determines that he can't send an ACK.
>
>The sender, then, queues up another send(), but this is sitting in a local buffer, waiting for an ACK, because there is previous data that hasn't yet been acknowledged, and there is not a full segment to send.  The receiver is similarly waiting, as we said, and after 200ms will finally send the ACK, that wakes up the sender.
>
>Note that we haven't got to the point of generating any data that the initial sender could receive using its recv() call, so what we've discussed so far is very applicable to your situation.

More details are available in [Knowledge Base Article 214397][aaag]

[Back to top](#toc)

---

## Sockets thread-safety
{: #thread_safe}

**Problem**

Are socket operations thread-safe? Can I call `send()` from two threads simultaneously? Can I call `send()` from one thread and `recv()` from another? Can I call `closesocket()` while another thread is reading/writing data?

**Solution**

Yes and no. The colloquial "socket" is not a single thing but rather a combination of:

* The socket object. This is a collection of whatever internal kernel- and user-mode data structures the OS is using to implement a "socket". These data structures are not accessible directly but only through sockets API (`recv()`, `bind()`, etc.)
* The socket handle. This is the variable of type `SOCKET` that works as an opaque reference to the socket object.
* For `SOCK_STREAM` sockets: the read and write data streams associated with it.

When talking about sockets and thread safety we must be careful which of the above concepts is currently under discussion. 

Let's start with socket objects. Here things are simple. Socket objects themselves are thread-safe which means that you can call any socket API from any thread while another one is calling another API. Doing this will not corrupt the internal data structures or cause your application to crash. In particular `closesocket()` may be called from any thread regardless of what other threads are doing with the socket (but see below!).

However, this is only part of the story. Now consider the socket handle. Unlike the socket object, the handle is the caller's variable and it is your responsibility to ensure that the handle refers to a valid socket object. It helps to think of the handle as a sophisticated pointer to the socket object. Just like pointer the handle could be dangling, i.e. referring to non-existent socket or, worse to a socket different from the one you intend it to point. Imagine the following scenario. Thread 1 is using a socket handle that has a value of 42. This handle currently refers to Socket Object 1. At the same time Thread 2 calls `closesocket()` on this handle. The Socket Object 1 is destroyed and the handle variable now contains "dangling" value 42 that refers to no existing socket. In the "best" scenario Thread 1 will continue to use the handle, passing it to some socket API which will fail with some error indicating that the handle is invalid. Why is this scenario "best"? Because imagine that some completely unrelated Thread 3 creates a new Socket Object 2 right after Thread 2 closed Socket Object 1. It is possible that this new socket object will also be assigned the same numeric handle value 42. At this point the original handle starts to refer to a completely unrelated Socket Object 2! The Thread 1 will happily run forward using a socket object completely different from the one it was using a moment ago. The results of such scenario could be much more disastrous than those of the "best" one.

Now it is true that modern operating systems often try to delay handle reuse to prevent such situations but you cannot reliably count on it. The conclusion is fairly simple: don't close any socket which is currently used by any other thread. This is, of course, not surprising. Replace "close socket" with more general "destroy object" and this is just the simple, most basic rule of any multi-threaded programming.

So how do you interrupt a `select()` loop? This is a complicated issue and it deserves [it own FAQ entry](#close_select).

Finally when working with stream sockets (such as used with TCP) you should also be with the safety of the data stream. And here the picture is different. The TCP stream, as most other streams in programming, is not thread safe. Consider what would happen when one thread writes "abcd" to a stream (be it TCP, stdio or IStream), while another writes "efgh". The stream object (socket, FILE * or IStream *) itself will usually be ok since it is thread safe but the data read from another end could be any combination of characters like "abefcdgh", "aefgbcdh" etc. So, practically, you do have to synchronize writes, and similarly, reads to/from a stream sockets.

Note that this also applies when you use IOCP reads though with a twist. When using IOCP you don't actually read or write data but rather queue requests to do so. Each request is queued in a FIFO order so it may appear that the OS will automatically ensure that the data sent from or received by your application is serialized correctly without you having to do anything. However, this is not so. Consider how your code will actually interact with IOCP. For reads the threads in your thread pool will be awakened when GetQueuedCompletionStatus() returns. At this point your thread will want either to directly process the data or to append it to some partly filled buffer. But between the return from `GetQueuedCompletionStatus()` and your doing anything, another thread in the pool may also return from `GetQueuedCompletionStatus()`. At this point you have a race condition. The two threads will compete over who processes the data first. Unless you do something the information about which request was queued first is completely lost to your code. Another way to look at this is to realize that IOCP simply pushes management of the communication stream to you rather than do it internally inside Winsock. To maintain the stream you will have to label your IOCP requests with sequence numbers or arrange buffers you give to IOCP in some sort of doubly linked list. In both cases you will still have to synchronize either sequence number increments or linked list updates between threads. The good news is that doing so is usually cheap and can be accomplished through Interlocked API rather than using full blown synchronization objects.

Now let's look at IOCP writes. Here again you will likely issue write requests in response either of completed read or another completed write. As long as you have more than one thread in your pool the different threads may arrive to the point of issuing a write request simultaneously. Unless you somehow synchronize them and their access to the source of the data they will either write the same piece twice or send pieces out of order. Again you can look at the situation as having you maintain the data stream and supply sequential chunks of it to Winsock.

The bottom line is: whenever you read from or write to a stream you will have to synchronize some parts of your code in one way or another.

[Back to top](#toc)

---

## Terminating a select() loop from another thread
{: #close_select}

**Problem**

How do I immediately terminate a select() loop from another thread if I cannot use `closesocket()`?

**Solution**

As was explained above calling `closesocket()` from another thread to force `select()` to exit is a bad idea. The correct and robust way to solve this problem is more complicated.

You probably had encountered this problem because you tried to write portable socket code. The `select()` functions seems to be a good solution in such a scenario because it seemingly allows one to perform non-blocking I/O in a platform independent way. Unfortunately this impression is not correct. The problem is that the subset of Posix `select()` functionality available on Windows makes it very much useless.

The Posix `select()` has two important features that are not available on Windows. One is that it works for any kind of file, not just sockets. Thus, you can simultaneously wait on a socket, a pipe and a file I/O from a single `select()` call. The second feature is that Posix `select()` can be interrupted by a signal. (If you don't know what a signal is and write portable code, stop doing what you are doing and read about them right away). These two features make interrupting Posix `select()` relatively simple. First you can add additional dummy pipe to the set of files to wait for and then write something to the pipe to make `select()` exit right away. Second you can interrupt it by sending your process a signal (since the usual need to terminate `select()` loop is on program termination which is caused by a signal to begin with, this functionality often comes for free).

All this is relatively straightforward but it makes the code using these techniques completely non-portable to Windows. But wait, if the code using `select()` is going to be non-portable anyway why waste time using it on Windows to begin with? The Windows platform already includes powerful asynchronous I/O primitives that allow you to do things `select()` could never do. In other words you need to change your approach. Instead of trying to write portable code using `select()` you will have to write two versions of your I/O loop, one for Posix and one for Windows. Yes, it is more work but unfortunately there isn't any other way. If you do it right you can isolate the platform dependent parts so that the rest of the code will not be aware of the differences and stay portable.

How do you write a waiting loop that can be terminated immediately on Windows? There are many approaches but the classic one is roughly the following

* Use some form of asynchronous I/O that requires you to wait on some HANDLE(s).
* Create an additional event object.
* Use `WaitForMultipleObjects()` or similar to wait on both the I/O HANDLE(s) and the event above.
* When you want to terminate the loop signal the event. The looping code should exit when this happens.

Note that this approach closely parallels the one used on Unix with an additional pipe in `select()` (the pipe works much like an event)

[Back to top](#toc)
