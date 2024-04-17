---
layout: post
title:  "*Fast* HTTPS server on Arduino Nano RP2040"
description: "How make a fast HTTPS or any other SSL/TLS server on Arduino Nano RP2040 Connect or other Arduino microcontrollers"
tags: embedded RP2040 Arduino 
---

<!-- References -->
[prev-art]: {% post_url 2024-04-03-https-arduino-nano-rp2040 %} 
[mbed-tls]: https://www.trustedfirmware.org/projects/mbed-tls/
[bear-ssl]: https://www.bearssl.org
[mbed-tls-v3.6.0]: https://github.com/Mbed-TLS/mbedtls/releases/tag/v3.6.0
[arduino-bear-ssl]: https://github.com/arduino-libraries/ArduinoBearSSL
[mbed-nano-tls]: https://github.com/gershnik/MbedNanoTLS
[mbed-example]: https://github.com/gershnik/BetterWiFiNINA/blob/mainline/examples/WiFiMbedTLSServer/WiFiMbedTLSServer.ino
[bear-example]: https://github.com/gershnik/BetterWiFiNINA/blob/mainline/examples/WiFiBearSSLServer/WiFiBearSSLServer.ino
[bear-usage]: https://www.bearssl.org/api1.html

<!-- End References -->

* TOC
{:toc}

## Intro

In the [previous article][prev-art] I showed how to make create an HTTPS server on Arduino Nano RP2040 using built-in platform [Mbed TLS][mbed-tls] implementation. It works, and works well but it is slooooooow.

The most expensive step in TLS session is the initial "handshake". This is when client and server use the certificate(s) and public/private key pair to agree on fast symmetric key to use. This step is the main overhead of an HTTPS session over plain HTTP one. By comparison, the symmetric encryption and other overhead on the rest of the session are usually pretty negligible.

In my measurements I got the following handshake durations for built-in Mbed TLS on Nano RP2040 with board package 4.1.1. For the certificate with:

- 1024 bit RSA key: ~2.6 second 
- 384 bit ECC key: ~3.2 seconds

The rest of the session added only <50ms on top of that. As you can see while not being prohibitively slow such overhead makes pervasive use of SSL all but impossible.

Can this situation be improved? Turns out it can, if you don't mind using a newer version of [Mbed TLS][mbed-tls] or a different SSL library altogether: [Bear SSL][bear-ssl].

(Note: there might be other libraries out there with better performance or better tradeoffs but these are the two I settled upon due to their excellent licensing terms)

## Approach 1: Newer Mbed TLS

First thing, if you haven't read the [previous article][prev-art] you should probably do so now. It explains how Mbed TLS can be used from Arduino and all the other stuff you need to know in order to understand the details below.

I am not sure what exact version of Mbed TLS is included with the current release of Mbed OS board package for Arduino - there appear to be no versioning macros in it - but it seems pretty old. An obvious thing to try is then to use the latest and greatest version ([3.6.0 LTS][mbed-tls-v3.6.0] at the time of this writing) and see if this improves matters. 

Trying to integrate a new version of Mbed TLS quickly hits a wall, however. Its symbols clash with the ones from the old version in the board package. Unlike some other libraries that commonly encounter this problem (like ICU, for example) Mbed TLS doesn't have provisions in its source code to add some custom prefix to its symbols. 

There are various possible ways to deal with this issue depending on your circumstances but I wanted to have something reusable that could be easily added to any project with minimal fuss. To this end I created [MbedNanoTLS][mbed-nano-tls] Arduino library. It simply repackages original Mbed TLS with appropriate configuration for Arduino Mbed OS and renaming its linker symbols (via `#pragma redefine_extname` if you are curious). It also adds `-O3` compilation switch to all its source files which turns out to be important.

Normally, Arduino code is complied with `-Os` (minimize size) compiler flag. Even when built like this the new Mbed TLS vastly outperformes the built-in one (by a _factor_ of 2!). However, turns out that Mbed TLS also benefits a lot from additional optimization. Building it with `-O3` (full aggressive optimizations) improves performance by an additonal 50%! I am guessing that Mbed TLS code contains lots of small functions that can greatly benefit from an aggressive inlining but this is just a guess - I haven't look in depth into this.

One additional change [MbedNanoTLS][mbed-nano-tls] does to Mbed TLS configuration compared to the stock one is to enable [MBEDTLS_AES_ROM_TABLES](https://mbed-tls.readthedocs.io/en/latest/kb/how-to/reduce-polarssl-memory-and-storage-footprint/) flag. This is under the assumption that RAM access is faster than memory mapped flash and the memory savings aren't that significant on Nano RP2040. I haven't done a detailed comparisons with and without the flag but it seems a good idea performance-wise. 

Armed with this library it becomes trivial to adjust the code from the [previous article][prev-art] to the new version. There are only 2 incompatibilities that need to be addressed.

Firstly, the new version does not rely on predefined `mbedtls_platform_std_nv_seed_read` and `mbedtls_platform_std_nv_seed_write` calls. Instead you provide your own callbacks, named however you want and register them. For example:

```cpp
#include <mbedtls/platform.h>

int nv_seed_read(uint8_t * buf, size_t buf_len) {
    ... same as before ...
}

int nv_seed_write(uint8_t * buf, size_t buf_len) {
    ... same as before ...
}

if (auto ret = mbedtls_platform_set_nv_seed(nv_seed_read, nv_seed_write); ret != 0) {
    while (true)
        printf("mbedtls_platform_set_nv_seed returned %x\n", res);
}
```

Second the call `mbedtls_pk_parse_key` has two additional parameters now so instead of

```cpp
mbedtls_pk_parse_key(&g_pkey, (const uint8_t *)g_privateKey,
                     sizeof(g_privateKey), nullptr, 0)
```

you need to say

```cpp
mbedtls_pk_parse_key(&g_pkey, (const uint8_t *)g_privateKey,
                     sizeof(g_privateKey), nullptr, 0,
                     nullptr, nullptr)
```

This is pretty much all. Otherwise the code from the previous article should run unachanged.

We can simplify things a bit further, though. The SSL socket reading and writing via custom BIO is a typical boilerplate copy/paste code that begs to be encapsulated into some wrapper and never be thought about again. To this end I added `WiFiMbedTLSSocket` class to `BetterWiFiNINA` library. You can see a fully functioning example of a web server using it [over here][mbed-example]. This is the same server I've been demonstarting here, only written using `WiFiMbedTLSSocket` class. It also allows you to use either built-in Mbed TLS like the previous article or [MbedNanoTLS][mbed-nano-tls] as discussed here.

With this code I get the following results: 

- 1024 bit RSA key: ~1.3 seconds 
- 384 bit ECC key: ~1 seconds

While both cases show huge improvement it is interesting to note that ECC case seem to benefit more than the RSA one.

Now let's look at a different approach.

## Approach 2: Bear SSL

[Bear SSL][bear-ssl] is a newer TLS library that is written from scratch with embedded systems in mind. It offers a different set of tradeoffs compared with Mbed TLS (more on this later) so it is interesting to try it. Most significantly it has far fewer requirements on the host platform and can be used on pretty much any Arduino board not just Nano RP2040.

Integrating this library is a breeze. There is already [ArduinoBearSSL][arduino-bear-ssl] library that did all the heavy lifting for us. It seems to contain most recent Bear SSL code (at the time of this writing) so there is no reason to try to do it yourself.

Unlike Mbed TLS, Bear SSL seems insensitive to compiler optimization level. Its performance appears the same under `-Os` and `-O3`. It looks like its code is heavily hand-optimized and there isn't much left for the compiler to do there other than to not make it worse.

On the less positive side, Bear TLS seems to contain some kind of undefined behavior bugs in its code. It works perfectly fine when built with stock GCC for Nano RP2040 but trying to use a newer compiler (GCC 13 in my case, that works perfectly fine with everything else) causes it to crash in many of its own initialization functions. It seems that there is an undefined behavior in it that causes misaligned memory access but I haven't looked in great detail. I am sure the bug will be found and fixed one day but in the meanwhile using newer compiler is impossible.

Where Bear SSL gets much more complicated is in doing I/O, specifically non-blocking I/O. Usually you don't want any blocking calls on Arduino to keep your `loop()` function available for other tasks. Writing generic non-blocking read and write functions for Bear SSL is not _terribly_ complicated but not easy either with many little tedious details to take care of. See "Generic I/O" section of [Bear SSL documentation][bear-usage] for gory details.

To make it usable and re-usable I added `WiFiBearSSLSocket` class to `BetterWiFiNINA` library that does all the heavy lifting and exposes a simple interface to client code. Using this class your main I/O code would look almost the same as with `WiFiMbedTLSSocket` (or with plain HTTP over `WiFiSocket` for that matter).

Another area where Bear SSL is more complicated is in parsing and loading certificates. Unlike other SSL libraries it doesn't give you "load certificate/private key from PEM" functions. Instead you need to use lower-level facilities to parse PEM, feed the parsed data into cetificate parser etc. all in a resumable step by step looping fashion. There is a good reason for that complexity - it allows client code much greater control over memory usage and resumable parsing. But power comes at a cost of complexity.

As an aside, what Bear SSL really wants you to do is to pre-parse the PEM into C data structures using a command line tool and include the result into your code. This however, adds another dimension of complexity and is not a good idea if you want your certificates to be replacible at runtime.

In any event you can see the same server we've been talking about implemented with Bear SSL in [this example][bear-example]. As you can see the main I/O loop is almost identical to Mbed TLS example and the difference, as expected, lies in initialization code. The general outline of initialization is the same though: load ceritificate, load private key, set up entropy, configure supported ciphers etc. so hopefully the code will not be too diffcult to follow.

Using this server I get the following results:

- 1024 bit RSA key: ~700 **milli**seconds
- 384 bit ECC key: ~1.6 seconds

As you can see there is again a huge improvement over built-in Mbed TLS but now the tradeoff is different. RSA clearly shines while ECC not so much and lags behind the new Mbed TLS.

## Which one to use?

As you can see both libraries provide much, much improved performance over the original approach with built-in Mbed TLS.
Either of them can be used effectively and the choice of which to use comes down to various tradeoffs. 

### Performance

Most of the world seem to be moving towards ECC certificates. They offer superieor strength at much lower key size which is especially important on embedded systems. Based on performance on ECC certificates alone Mbed TLS is a clear winner. Unless you have a reason to use RSA certificates, that is, in which case Bear SSL is actually faster. Keep in mind that the performance might change as both libraries evolve. It is possible for the advantage to flip in some future version.

### Memory usage

We haven't been talking about memory usage much before but it is also obviously an important consideration. Full, unconstrained, HTTPS server requires about 40kB of memory _per connection_. For a very good explantion why see the section "I/O Buffer" in [Bear SSL documentation][bear-usage]. Keep in mind that the total memory usage is more than just an I/O buffer though. Both Bear SSL and Mbed TLS will use about that amount by default. 

One thing to keep in mind here is that while Mbed TLS dynamically allocates its memory and Bear TLS allows you to use static buffers this makes no difference to the total memory consumption - both come from the same source: RAM. Dynamic memory allocation can have negative consequences because of memory fragmentation unless you are careful but in practice for a single threaded web server where per-connection data is all allocated and deallocated _en masse_ this is not a big concern. On the other hand dynamic allocation is somehwat friendlier in supporting multiple simultaneous connections though. You would have to do some non-trivial buffer management manually to do it with static buffers.

Where Bear SSL shines though, is in that it is possible to reduce that amount somewhat to something like 20kB by using a smaller buffer and sacrificing full-duplex functionality (usually not needed). On Nano RP2040 with 256kB of RAM this might or might not matter but on devices with less RAM this might be the most crucial factor. As far as I know nothing like this is possible with Mbed TLS but perhaps I wasn't looking deep enough.

### Portability

If Nano RP2040 is not your only target there is a differtence too. Bear SSL is trivially portable to any Arduino device (and more). In theory, Mbed SSL is also easily portable but in practice it means lots of work figuring out the correct configuration parameters, hookups into board APIs and compiler switches. Not a terribly complicated thing to do but still a hassle and a source of problems.

### Robustness and ease of use

This is an area in which Mbed TLS has so far a clear advantage. It's been around for a while, has a good documentation, developer friendly API and has appears to fix its vulnerabilites. Bear SSL, on the other hand is much newer, documentation is scarce and it's hard to figure out how to use it correctly. I have already mentioned that it has trouble under GCC 13 (at least on Cortex M0). Its API is very powerful but requires much more experience and attention to use.

At the end the decision of which library to prefer comes down to you specific set of circumstances and your application requirements. There is, in my mind no overall clear winner here.

## Other performance related things

### SSL session caching

As long as you have sufficient memory you should enable SSL caching. For Mbed TLS this is done via `mbedtls_ssl_cache_init` and `mbedtls_ssl_conf_session_cache` calls. For Bear SSL this is done via `br_ssl_session_cache_lru_init` and `br_ssl_server_set_cache` calls. For Mbed TLS you might want to also limit cache duration since the default - 1 day might result in too big memory consumption for an embedded device.

Session caching removes the need to perform a full handshake reducing the handshake duration to almost nothing. This makes HTTPS session almost as fast as plain HTTP one. Keep in mind that both server and client must support caching for this to work.
Also keep in mind, that the cache only helps with _repeated_ connections from the same client. A new, non-cached connection will still pay the full handshake price. 

### Probably useless: use ECCX08 chip

Nano RP2040 comes equipped with ATECC608A chip that _in theory_ could be used to speed up SSL implementation. In practice, trying to use it for this purpose hits numerous problems which makes it, in my opinion, worthless. Specifically:

- In order to use that chip for anything you have to **irreversibly** lock its configuration. Unless you have very many boards to brick this precludes any type of experimentation and tweaking.
- It only supports a single elliptic curve (it doesn't do RSA at all as far as I can see, in case you care) - SECP256R1. Which is pretty old and as far as I can tell not used much anymore. If your certificate uses a different elliptic curve the chip won't help you to speed up SSL handshake at all. As far as I can tell it doesn't support RSA assymetric encryption at all.
- While it can speed up symmetric encryption and random number generation none of these are performance issue with SSL on Nano RP2040 to begin with. In addition any communication with this chip has to go through multiple stages with various `delay(...)` between them. Due to this I doubt it will be able to outperform the CPU doing the algorithm directly. Perhaps this might be different for other boards.

It seems that the chip's primary and only use case is to serve as a "secure enclave" and not a general purpose crypto accelerator. 

If you have other experiences with it and were able to make it work and speed things I'd be very curious to know.

## Conclusion

That's pretty much it. With current state of technology we can have an HTTPS server on Nano RP2040 that can respond in a ~1 second + change to a brand new client request. This is quite usable though not anywhere near speeds you can get on more powerful platforms.

All the code you need to make it work is located in these two samples:

- [Mbed TLS Sample][mbed-example]
- [Bear SSL Sample][bear-example]










