---
layout: post
title:  "Loading Android ART virtual machine from native executables"
tags: android jni c++
---

* TOC
{:toc}

## Intro

This article is a summary of what it takes to create a native Android command-line application that loads and uses ART virtual machine. The precise details on how to do it have changed over the years as Android evolved. This post is accurate as of Android 11 and NDK 21.3. 

The self-contained source code for this article can be found on [Github](https://github.com/gershnik/android-cmdline-jni)

I assume that if you are reading this you have some specific need to run native command line code (as opposed to normal Java app that loads native code). In my experience this is usually caused by desire to run cross platform unit or performance tests in the same way you do on desktop. While it is possible to re-package the tests into a shared library that is loaded from Java/Kotlin APK this often add a huge amount of build system and CI/CD complexity. Running the code in the same way as on desktop is often simpler.

The particular need to not just have a command line executable but also load Java into it stems from the fact that the code being tested often has Android specific functionality (wrapped into `#ifdef __ANDROID___`) that requires JNI presence on Android. 


## The basics: running command line executable on Android

Let's start with a simplest possible executable - the famous "Hello, World!" application. Create a `main.cpp` that looks like this:

```cpp
//main.cpp
#include <iostream>

int main() {
    std::cout << "Hello, World!\n";
}
```

To build it for Android let's start with the following `CMakeLists.txt` file

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.16)

set(CMAKE_CXX_STANDARD 17)

project(hello)

add_executable(hello
    main.cpp
)

```

and build it like this:

```bash
mkdir out
cd out
cmake -DCMAKE_BUILD_TYPE:STRING=Debug \
      -DCMAKE_TOOLCHAIN_FILE:FILEPATH=$NDK_DIR/build/cmake/android.toolchain.cmake \
      -DANDROID_ABI:STRING=arm64-v8a \
      -DANDROID_PLATFORM:STRING=19 \
      -DANDROID_STL:STRING=c++_static 
      -H .. 
      -B . 
      -G "Unix Makefiles"
make
```

Where `NDK_DIR` is the location of your NDK (usually `$ANDROID_HOME/ndk/major.minor.build` these days).
Note that the ABI is set to `arm64-v8a` to run on a physical 64-bit device. If you want to run on a simulator use `x86` or `x86_64`. If for some reason you have an old 32-bit device use `armeabi-v7a`.

If everything is ok, the above should produce `out\hello` executable. Now we need to put it on the device and run. In the past this would have been simple - just push it into `/sdcard` directory and run from there but Android has been tightening security screws relentlessly and nowadays most user-accessible places on the filesystem do not allow code execution. If your device is rooted or you are using an emulator this is not a big deal - you can override prohibitions or put the executable where execution is allowed. For example

```bash
adb root
adb shell mkdir -p /data/hellodir
adb push hello /data/hellodir
adb shell /data/hellodir/hello
```

However, if you device is not rooted the above won't work. Instead you have to use one location that, at the time of this writing, is left available: `/data/local/tmp`. Incidentally Android Studio uses that location to put helper files for remote debugging in, so presumably it is not an oversight and the place should stay available at least for a while. 
So, instead of the above, do:

```bash
adb shell mkdir -p /data/local/tmp/hellodir
adb push hello /data/local/tmp/hellodir
adb shell /data/local/tmp/hellodir/hello
```

If it works, you should see "Hello, World!" printed as an output and are done with the basics.

## Loading Java 

The overall process is conceptually simple:

* Load shared library containing ART virtual machine into the process.
  
  The shared library containing ART VM is called `libart.so`. In the past it was located where all normal system shared libraries live: `system\lib`. Now it seems to be moved into `/apex/com.android.art/lib` or `/apex/com.android.art/lib64`. (If you are on Android 10 it is `/apex/com.android.runtime/lib[64]` there)

  Loading the library via `dlopen()` from the new location is not enough, though. It has implicit dependencies for other libraries in the same location which cannot be resolved in this case. Instead, you need to set `LD_LIBRARY_PATH` for the process.
* Call `JNI_CreateJavaVM` [invocation API](https://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/invocation.html)
  
  The API is not exposed by Android JNI headers but it exists. You will need to declare it yourself.

* Exporting function(s) expected to be provided by main executable. 
  
  The details of this also changed over time. See below for current state of affairs.

The updated `main.cpp` is as follows:

```cpp
//main.cpp

#include <iostream>


#if defined(__ANDROID__)

#include <dlfcn.h>
#include <jni.h>

extern "C"
{
    typedef int JNI_CreateJavaVM_t(void *, void *, void *);

    __attribute__((visibility("default"))) void AddSpecialSignalHandlerFn() 
    { }
}

static auto load_art() -> std::pair<JavaVM *, JNIEnv *>
{
    JavaVMOption opt[] = {
        { "-Djava.library.path=/data/local/tmp/hellodir", nullptr },
        //{ "-verbose:jni", nullptr } uncomment if you want to see JNI debugging info in logcat
    };

    JavaVMInitArgs args = {
        JNI_VERSION_1_6,
        std::size(opt),
        opt,
        JNI_FALSE
    };

    void * libart = dlopen("libart.so", RTLD_NOW);
    if (!libart) 
    {
        std::cerr << dlerror() << std::endl;
        abort();
    }

    auto JNI_CreateJavaVM = (JNI_CreateJavaVM_t *)dlsym(libart, "JNI_CreateJavaVM");
    if (!JNI_CreateJavaVM)
    {
        std::cerr << "No JNI_CreateJavaVM: " << dlerror() << std::endl;
        abort();
    }

    std::pair<JavaVM *, JNIEnv *> ret;
    int res = JNI_CreateJavaVM(&ret.first, &ret.second, &args);
    if (res != 0)
    {
        std::cerr << "Failed to create VM: " << res << std::endl;
        abort();
    }
    return ret;
}

#endif

int main() {
    #if defined(__ANDROID__)

        auto [vm, env] = load_art();
        
        jclass systemCls = env->FindClass("java/lang/System");
        jfieldID outField = env->GetStaticFieldID(systemCls, "out", "Ljava/io/PrintStream;");

        jclass printStreamCls = env->FindClass("java/io/PrintStream");
        jmethodID printMethod = env->GetMethodID(printStreamCls, "print", "(Ljava/lang/String;)V");
        
        
        jobject outObj = env->GetStaticObjectField(systemCls, outField);
        const char16_t text[] = u"Hello, World from Java!\n";
        jstring jtext = env->NewString((const jchar *)text, std::size(text) - 1);
        env->CallVoidMethod(outObj, printMethod, jtext);

    #else
        std::cout << "Hello, World!\n";
    #endif
}

```

The empty `AddSpecialSignalHandlerFn` function is necessary. It is something [libsigchain](https://android.googlesource.com/platform/art/+/refs/heads/master/sigchainlib/) appears to require. If the function isn't present it aborts the process. 
In order for the function to be visible to external code you also need to pass `--export-dynamic` flag to the linker. By default symbols from executable aren't exported. So modified `CMakeLists.txt` should look like:

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.16)

set(CMAKE_CXX_STANDARD 17)

project(hello)

add_executable(hello
    main.cpp
)

target_link_options(hello PRIVATE

    "$<$<PLATFORM_ID:Android>:-Wl,--export-dynamic>"
)
```

Finally, to run it you need to supply LD_LIBRARY_PATH

```bash

adb shell mkdir -p /data/local/tmp/hellodir
adb push hello /data/local/tmp/hellodir
adb shell LD_LIBRARY_PATH=/apex/com.android.art/lib64 /data/local/tmp/hellodir/hello

```

Replace `lib64` with `lib` if building for 32-bit.
That's all. As mentioned above you can find the source code at [Github](https://github.com/gershnik/android-cmdline-jni)







