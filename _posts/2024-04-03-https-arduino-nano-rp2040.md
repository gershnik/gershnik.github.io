---
layout: post
title:  "HTTPS server on Arduino Nano RP2040"
description: "How to run HTTPS or any other SSL/TLS server on Arduino Nano RP2040 Connect or other MBED based microcontrollers"
tags: embedded RP2040 Arduino 
---

<!-- References -->
[board]: https://docs.arduino.cc/hardware/nano-rp2040-connect/
[nina-chip]: https://www.u-blox.com/en/product/nina-w10-series-open-cpu
[nina-firmware]: https://github.com/arduino/nina-fw
[wifinina]: https://github.com/arduino-libraries/WiFiNINA
[better-wifinina]: https://github.com/gershnik/BetterWiFiNINA
[socket-pr]: https://github.com/arduino-libraries/WiFiNINA/pull/278
[mbedos-arduino]: https://docs.arduino.cc/software/ide-v1/tutorials/getting-started/cores/arduino-mbed_nano
[mbed-tls]: https://www.trustedfirmware.org/projects/mbed-tls/
[mbed-tls-nvram]: https://mbed-tls.readthedocs.io/en/latest/kb/how-to/how_to_integrate_nv_seed/
[crosstalk]: https://github.com/arduino-libraries/WiFiNINA/issues/176
[no-client-and-server]: https://www.reddit.com/r/arduino/comments/exc3hd/wifinina_server_and_client_simultaneously/
[rp2040-entropy]: https://www.i-programmer.info/programming/148-hardware/17030-master-the-pico-wifi-random-numbers.html
[wifinina-webserver-sample]: https://docs.arduino.cc/retired/library-examples/wifi-library/WiFiWebServer/
[gist]: https://gist.github.com/gershnik/072d112071d0d61adc495d6f36947d45

<!-- End References -->

* TOC
{:toc}

## The problem

[Arduino Nano RP2040 Connect][board] comes equipped with [uBlox Nina W-102 module][nina-chip], which, at least when using the its [default firmware][nina-firmware] only supports **client** TLS connections. 

Accordingly the [WIFINina][wifinina] library - the standard interface to the Nina module - also only exposes `WiFiSSLClient` class and there is no `WiFiSSLServer` one.

So, on the surface, it would appear that short of writing your own Nina firmware (which would be a very large undertaking) it is impossible to run an HTTPS or any other TLS server on Nano RP2040.

Appearances are deceiving though. Turns out it is quite easy to create such a server. It won't win any speed awards and will be much more memory demanding that a plain HTTP one but it will work well and allow you to use HTTPS at least for some remote calls where it matters.

## Why use HTTPS?

If you have found this article you probably already have a good reason why. In my case I needed HTTPS to pass secrets: passwords and such to and from the software running on the board. Rather than giving up and sending them over plain HTTP as many IoT gadgets unfortunately do I decided to see if something better is possible.

## The general idea

Nano RP2040 software is [Mbed OS based](https://docs.arduino.cc/software/ide-v1/tutorials/getting-started/cores/arduino-mbed_nano). Mbed OS includes [Mbed TLS][mbed-tls] which, incidentally, is the same library NINA firmware uses to provide client SSL functionality. Mbed TLS, following in the footsteps of OpenSSL, allows one to use custom "BIO" (binary I/O) instead of hooking up directly to OS sockets. Thus, at least in theory, it should be possible to hook up Mbed TLS to plain TCP communications via [WIFINina][wifinina]'s `WiFiServer` class and run a TLS server on Nano RP2040!

Trying to implement this idea hits some complications though.

## Complications 

### 1: WIFINina 

WIFINina exposes network communications via a "simplified" (or dumbed-down, if you will) interface. It seems to be intended to make a beginner's sample easy to write, but, as a result, it makes it impossible to write anything more sophisticated than a beginner's sample. In particular, you cannot have multiple TCP servers and you cannot mix TCP server and client code. If you attempt either their communications will get all mixed up resulting in "crosstalk". (See for example [this][crosstalk] and [this][no-client-and-server]). 

It might be possible to hook up WIFINina's WiFiServer interface as Mbed TLS BIO to produce a very simple TLS "sample" but, realistically, if you intend to use HTTPS you will want it alongside a regular HTTP server - not as a sole means of communication. Given that WIFINina cannot handle multiple servers this is a non-starter.

Fortunately, the actual Nina firmware, starting from version 1.5 provides an almost normal plain BSD socket interface. It's just that WIFINina library doesn't expose it to you. 

This is, however, easy to rectify. Download [BetterWIFINina][better-wifinina] library and add it to your Arduino libraries. This is a WIFINina fork I created that adds support for plain BSD sockets to WIFINina via a new `WiFiSocket` class. (This change has also been [submitted to WIFINina][socket-pr] but when and if it will be added is unknown at the time of this writing).

Armed with this library you can now easily have multiple HTTP and HTTPS servers in your application as well as do client TCP calls without any issues. 

Now we need to connect it to Mbed TLS and face the next problem.

### 2: Source of entropy

Mbed TLS uses a random number generator to do its cryptographic magic and a random number generator needs a source of randomness aka entropy. What source of randomness to use and how to hook up to it is determined at Mbed TLS configuration/build time. It so happens that the version bundled with Arduino board package expects so called [NVRAM source of entropy][mbed-tls-nvram].

What it means is that Mbed TLS expects you, the user, to provide callbacks that will:

- load a random seed from somewhere (NVRAM) 
- save updated seed in the same place 

Note that the seed should start from something truly random. Otherwise, the entire sequence of random numbers it will internally generate can be reconstructed.

From observation the size of the random seed Mbed TLS asks for is 64 bytes and it reads and updates it only once upon initialization.

It all sounds simple but the problem is, of course, that Nano RP2040 Connect does not have NVRAM. It does have flash memory but writing the seed into it on every startup is a bit fishy. Flash wears out and even though startups usually aren't frequent still for a long living device this is a concern.

Fortunately, there is a simpler way. RP2040 chip does have a **real** source source of entropy from a Ring Oscillator (ROSC). Mbed TLS is just configured conservatively for a general Arduino device and assumes that there isn't one. 

How to properly use RP2040 random number generator is helpfully documented [here][rp2040-entropy].

With a real source of entropy it is simply possible to provide "fake NVRAM" to Mbed TLS in the following way:
- Use a memory buffer and fill it in with real random data on startup.
- Provide it to Mbed TLS as fake NVRAM when it asks for a seed to read.
- Write whatever Mbed TLS wants to write in the same buffer but _don't persist it_.

This ensures that Mbed TLS gets a truly random seed on every startup and avoids the need to persist anything.

### 3: Certificate algorithm and size

Since you want to run an HTTPS server you will need a certificate. Whether you are going to generate a self-signed one or obtain an official one from LetsEncrypt or somewhere else turns out that the algorithm of the certificate may be an issue. 

On desktop nobody really pays attention to how big the actual certificates are. Who cares? On a microcontroller we only a have a few kilobytes of memory available and loading a really big certificate and private key might be an issue.

When I first tried to use Mbed TLS on Nano RP2040 I naively generated an RSA certificate with 8192 bits key size. This produced 7kB private key PEM file and 4kB certificate PEM. While _storing_ such data in your program isn't a problem - being immutable they will go into flash anyway - _loading_ them is a different story. Trying to load that private key by Mbed TLS failed - I suspect due to running out of memory trying to parse and process it but I didn't investigate deeply.

What worked successfully is using ECC algorithm instead. This algorithm uses much, much smaller keys. When choosing it my macOS certificate assistant gives me a choice of 256, 384 and 512 bits for private key. Using 384 bits key produces a certificate PEM of 1kB and private key PEM of 460 bytes. Both loads without an issue by Mbed TLS.

So, to conclude, use ECC algorithm to generate your own certificate. It seems that recent certificates from LetsEncrypt also use it so you should be good if you procure a certificate from there. If you obtain a certificate from other source you will need to figure out how to get an ECC one.

## Putting it all together

### Preliminaries

To get it out of the way you will need at least the following headers:

```cpp
//BetterWiFiNINA library
#include <BetterWiFiNINA.h>

//RP2040 ring oscillator
#include <hardware/structs/rosc.h>

//Mbed TLS
#include <mbedtls/entropy.h>
#include <mbedtls/ssl.h>
#include <mbedtls/ctr_drbg.h>
#include <mbedtls/x509.h>
#include <mbedtls/ssl.h>
#include <mbedtls/ssl_cache.h>
#include <mbedtls/net.h>
```


In addition, on MBed OS it is possible to directly `printf` to serial output rather than going through `sprintf`/`Serial.print` rigamarole. To enable this feature you need to put the following in your code.

```cpp
REDIRECT_STDOUT_TO(Serial) 
```

The rest of the code below will be using printf for diagnostic output

### Setting up entropy

```cpp
//Our NVRAM :)
uint8_t g_nvRandomSeed[64];

//Mbed TLS callbacks to access "NVRAM"
extern "C" {

    int mbedtls_platform_std_nv_seed_read(uint8_t * buf, size_t buf_len) {
        size_t toRead = std::min(buf_len, sizeof(g_nvRandomSeed));
        memcpy(buf, g_nvRandomSeed, toRead);
        return toRead;
    }

    int mbedtls_platform_std_nv_seed_write(uint8_t * buf, size_t buf_len) {
        size_t toWrite = std::min(buf_len, sizeof(g_nvRandomSeed));
        memcpy(g_nvRandomSeed, buf, toWrite);
        return toWrite;
    }
}

//From: https://www.i-programmer.info/programming/148-hardware/17030-master-the-pico-wifi-random-numbers.html
uint8_t getRandomByte() {
    uint32_t random = 0;
    uint32_t bit = 0;
    for (int k = 0; k < 8; k++) {
        while (true) {
            bit = rosc_hw->randombit;
            sleep_us(10);
            if (bit != rosc_hw->randombit)
                break;
        }  
        random = (random << 1) | bit; 
        sleep_us(10); 
    }
    return uint8_t(random);
}

//Mbed TLS entroy and random number generator objects
mbedtls_entropy_context g_entropy;
mbedtls_ctr_drbg_context g_ctr_drbg;

void setup() {
    ...
    
    //Initialize entropy and RNG
    for(auto & b: g_nvRandomSeed)
        b = getRandomByte();
    mbedtls_entropy_init(&g_entropy);
    mbedtls_ctr_drbg_init(&g_ctr_drbg);
    if (auto ret = mbedtls_ctr_drbg_seed(&g_ctr_drbg, mbedtls_entropy_func, &g_entropy,
                                         nullptr, 0); ret != 0) {
        while(true)
            printf("mbedtls_ctr_drbg_seed failed with %x\n", ret);
    }
    ...
}
```

### Setting up SSL configuration

Now we can finally get to setting actual SSL stuff set up. For that we need to set up SSL config and, most importantly load up server certificate and private key. Make sure you read the [section about certificate algorithm](#3-certificate-algorithm-and-size) before generating/obtaining the certificate. Once you have the certificate export it and the private key into PEM files, if they aren't in that format already. Then embed them into your code like this (this uses C++ raw string literals to allow you to copy/paste PEM content as-is):

```cpp
//Our certificate and private key. 
//NOTE: both must be NULL terminated strings
//even though the functions that loads them take
//size as an argument
const char g_certificate[] = 
R"(-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----)";

const char g_privateKey[] = 
R"(-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----)";
```

Now we can set up the SSL configuration:

```cpp
mbedtls_ssl_config g_ssl_conf;
mbedtls_x509_crt g_srvcert;
mbedtls_pk_context g_pkey;

void setup() {
    ...

    mbedtls_ssl_config_init(&g_ssl_conf);
    mbedtls_x509_crt_init(&g_srvcert);
    mbedtls_pk_init(&g_pkey);

    //load the certificate
    if (auto ret = mbedtls_x509_crt_parse(&g_srvcert, (const uint8_t *)g_certificate,
                                          sizeof(g_certificate)); ret != 0) {
        while(true)
            printf("mbedtls_x509_crt_parse failed with %x\n", ret);
    }

    //load the private key
    if (auto ret = mbedtls_pk_parse_key(&g_pkey, (const uint8_t *)g_privateKey,
                                        sizeof(g_privateKey), nullptr, 0); ret != 0) {
        while(true)
            printf("mbedtls_pk_parse_key failed with %x\n", ret);
    }


    //set up SSL config params
    if (auto ret = mbedtls_ssl_config_defaults(&g_ssl_conf,
                                               MBEDTLS_SSL_IS_SERVER,
                                               MBEDTLS_SSL_TRANSPORT_STREAM,
                                               MBEDTLS_SSL_PRESET_DEFAULT); ret != 0) {
        while(true)
            printf("mbedtls_ssl_config_defaults failed with %x\n", ret);
    }

    //connect our RNG to SSL config
    mbedtls_ssl_conf_rng(&g_ssl_conf, mbedtls_ctr_drbg_random, &g_ctr_drbg);

    //load certificate data into SSL config
    mbedtls_ssl_conf_ca_chain(&g_ssl_conf, g_srvcert.next, nullptr);
    if (auto ret = mbedtls_ssl_conf_own_cert(&g_ssl_conf, &g_srvcert, &g_pkey); ret != 0) {
        while(true)
            printf("mbedtls_ssl_conf_own_cert failed with %x\n", ret);
    }
}
```

### Running a TCP server

Now, with all the SSL configuration out of the way it's time to start a normal TCP server. If you've done any kind of socket programming before the following should look very familiar. For simplicity I am going to use a trivial one connection at a time blocking accept, non-blocking read/write server. It is of course possible, and in fact advisable, to use a fully non-blocking server with multiple connections but this will complicate this sample code needlessly.

```cpp

WiFiSocket g_sslServerSocket;

void setup() {

    ...

    //Create a socket
    g_sslServerSocket = WiFiSocket(WiFiSocket::Type::Stream, WiFiSocket::Protocol::TCP);
    if (!g_sslServerSocket) {
        while(true)
            printf("Creating server socket failed: error %d\n", WiFiSocket::lastError());
    }
    //Bind to port 443
    if (!g_sslServerSocket.bind(443)) {
        while(true)
            printf("Binding server socket failed: error %d\n", WiFiSocket::lastError());
    }
    //And start listening
    if (!g_sslServerSocket.listen(5)) {
        while(true)
            printf("Listen on server socket failed: error %d\n", WiFiSocket::lastError());
    }
}

void loop() {
    //accept an incoming connection
    IPAddress addr;
    uint16_t port;
    auto sslSessionSocket = g_sslServerSocket.accept(addr, port);
    if (!sslSessionSocket) {
        printf("Accept on server socket failed: error %d\n", WiFiSocket::lastError());
        delay(100);
        return;
    }
    //set the session socket to non-blocking
    if (!sslSessionSocket.setNonBlocking(true))  {
        printf("Setting socket to non-blocking failed: error %d\n", WiFiSocket::lastError());
        delay(100);
        return;
    }
}
```

Now we have a bare-bones server that will accept a connection on port 443 and immediately close it. On to the next step.

### Connecting session socket to Mbed TLS

This is the main part where everything finally comes together. We are going to connect Mbed TLS to the session socket via a custom BIO to be able to do actual reading and writing. First we need to set up BIO reading and writing callback functions. Both have a `void * context` first argument that is meant to allow us to pass information about where to perform reading/writing from. In our case the context will simply be the session socket.

The callbacks presented below are generic and can be used with either blocking or non-blocking sockets. They also demonstrate how to map socket errors to proper Mbed TLS error codes.

```cpp
int sendTLSCallback(void * ctx, const uint8_t * buf, size_t len) {
    auto socket = static_cast<WiFiSocket *>(ctx);
    auto sent = socket->send(buf, len);
    if (sent < 0) {
        auto err = WiFiSocket::lastError();
        if (err == EWOULDBLOCK)
            return MBEDTLS_ERR_SSL_WANT_WRITE;
        printf("writing to socket failed with error: %d\n", err);
        if (err == ECONNRESET)
            return MBEDTLS_ERR_NET_CONN_RESET;
        return MBEDTLS_ERR_NET_SEND_FAILED;
    }
    return sent;
}

int recvTLSCallback(void * ctx, uint8_t * buf, size_t len) {
    auto socket = static_cast<WiFiSocket *>(ctx);
    auto read = socket->recv(buf, len);
    if (read < 0) {
        auto err = WiFiSocket::lastError();
        if (err == EWOULDBLOCK)
            return MBEDTLS_ERR_SSL_WANT_READ;
        printf("reading from socket failed with error: %d\n", err);
        if (err == ECONNRESET)
            return MBEDTLS_ERR_NET_CONN_RESET;
        return MBEDTLS_ERR_NET_RECV_FAILED;
    }
    return read;
}
```

With these callbacks defined now we can connect them to the socket:

```cpp

//a simple RAII wrapper over mbedtls_ssl_context
struct SSLSessionContext : mbedtls_ssl_context {
    SSLSessionContext() { mbedtls_ssl_init(this); }
    ~SSLSessionContext() { mbedtls_ssl_free(this); }
    SSLSessionContext(const SSLSessionContext &) = delete;
    SSLSessionContext & operator=(const SSLSessionContext &) = delete;
};

void loop() {
    ...

    //declare and initialize SSL session context
    SSLSessionContext sslSessionContext;

    //connect session context to BIO
    mbedtls_ssl_set_bio(&sslSessionContext, &sslSessionSocket, sendTLSCallback, recvTLSCallback, nullptr);

    //and to the SSL config we set up before
    if (auto ret = mbedtls_ssl_setup(&sslSessionContext, &g_ssl_conf); ret != 0) {
        printf("mbedtls_ssl_setup returned %x\n", ret);
        delay(100);
        return;
    }    
}
```

And now the only thing remaining is to perform the actual reading and writing

### Reading and writing

Since proper parsing of HTTP is not the topic of this sample code I'll borrow the most atrocious HTTP parsing ever from the WIFINina's [WiFiWebServer sample][wifinina-webserver-sample]. 🙂 

Communicating using Mbed TLS is almost the same as using regular sockets except for one additional step: handshake.
Here is what it looks like:

```cpp
void loop() {
    ...

    //perform handshake
    while (true) {
        auto res = mbedtls_ssl_handshake(&sslSessionContext);
        if (res >= 0)
            break;
        if (res != MBEDTLS_ERR_SSL_WANT_READ && res != MBEDTLS_ERR_SSL_WANT_WRITE) {
            printf("TLS handshake error: %x\n", res);
            delay(100);
            return;
        }
    }

    uint8_t buffer[256];
    bool currentLineIsBlank = true;
    bool doneReading = false;

    //read until \n\r\n
    while(!doneReading) {
        auto read = mbedtls_ssl_read(&sslSessionContext, buffer, sizeof(buffer));
        if (read < 0) {
            if (read == MBEDTLS_ERR_SSL_WANT_READ)
                continue;
            printf("Read error: %x\n", read);
            delay(100);
            return;
        }
        for(int i = 0; i != read; ++i) {
            auto c = buffer[i];
            if (c == '\n' && currentLineIsBlank) {
                doneReading = true;
                break;
            }
            if (c == '\n') {
                currentLineIsBlank = true;
            } else if (c != '\r') {
                currentLineIsBlank = false;
            }
        }
    }

    //write response
    static const char response[] = 
            "HTTP/1.1 200 OK\r\n"
            "Content-Type: text/html\r\n"
            "Connection: close\r\n"
            "\r\n"
            "<!DOCTYPE HTML>\n"
            "<html><body>Hello World!</body></html>";

    size_t written = 0;
    while(written != sizeof(response)) {
        auto sent = mbedtls_ssl_write(&sslSessionContext, 
                                      (const uint8_t *)response + written, 
                                      sizeof(response) - written);
        if (sent < 0) {
            if (sent == MBEDTLS_ERR_SSL_WANT_WRITE)
                continue;
            printf("Write error: %x\n", sent);
            delay(100);
            return;
        }
        written += sent;
    }
}
```

And that's pretty much all there is to it. (Note that in theory you are also supposed to shutdown SSL connection cleanly before closing the socket but in practice none of the clients seem to mind if the socket is just closed abruptly.)

If you've done everything correctly this should produce a working HTTPS server.

## Conclusion

A fully functioning sketch containing all the code above can be found [at this gist][gist]. You will only need to supply your own WiFi password, certificate and private key to run it.

Obviously the code presented in this article is a sample. In real life you would want to encapsulate various pieces better but doing so is left as an exercise for a reader. 🙂

It should be possible to adapt this code to run on other MBed OS based Arduino chips with Nina module. You will only have to figure out what to do about the source of entropy. If you have one a trick similar to the one here will work. Otherwise, you you have NVRAM simply initializing a random seed there and giving it to MBed TLS will work equally well.


## License

The following license applies to all code on this page

```txt
Copyright (c) 2024, Eugene Gershnik
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

1. Redistributions of source code must retain the above copyright notice, this
   list of conditions and the following disclaimer.

2. Redistributions in binary form must reproduce the above copyright notice,
   this list of conditions and the following disclaimer in the documentation
   and/or other materials provided with the distribution.

3. Neither the name of the copyright holder nor the names of its
   contributors may be used to endorse or promote products derived from
   this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
```

