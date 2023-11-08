---
layout: post
title:  "Creating Android command line application in Java"
tags: android jni c++ java
---

* TOC
{:toc}

<!-- References -->

[prev-article]: 2021-03-26-load-art-from-native.md
[android-sdk]: https://developer.android.com/studio
[android-ndk]: https://developer.android.com/studio/projects/install-ndk
[sdk-manager]: https://developer.android.com/studio/intro/update#sdk-manager
[app_process]: https://android.googlesource.com/platform/frameworks/base/+/master/cmds/app_process/app_main.cpp

<!-- -->

## Intro

Some time ago I described how to [create a native Android command-line application that loads and uses ART virtual machine][prev-article]. But what if you want to create a Java command line application (that may or may not also load native code) and run it on Android?

Why would you want to do that? Same kind of reasons as in the previous article, basically. Perhaps you have some test harness or utility written in Java for desktop and want to adapt it to Android without drastically changing the code and build structure. Or perhaps you want to extract some information from Android device for automation or similar purposes, have the Java code to do so and just want to run it via `adb shell` rather than invent some elaborate mechanism to communicate with a 'real' Android app.

Whatever you reasons, running Java command line app on Android is easy (after all Android is all about Java).
Many of the concepts of how to do it were discovered long ago by other people[^1],[^2]. 
Some details changed since those articles were written so this is an attempt to describe the way to do it at the time of this writing.

## Preparations

I assume you already know how to create a _desktop_ command line Java application and package it in a `.jar` file.
This article uses Gradle in its examples but you can use any build system you like. The operations will be explained conceptually first before giving a Gradle example so it should be relatively easy to change them to Ant, Maven or something else.
If you use JNI I also assume that you already know how to create a shared library and load it from Java on desktop as well as what `java.library.path` is.

What you will need to make it all run on Android is [Android SDK][android-sdk] and, if you use native code, [NDK][android-ndk] which is available as one of the SDK components. 

In what follows Android SDK full directory path will be denoted by `<ANDROID_SDK>`. Replace as appropriate for your setup.

## Steps

The conceptual steps to make a command line Android app are simple

1. Build a normal JAR containing your code just like you do on desktop
2. Convert this JAR _and all of its runtime dependencies_ into a DEX file
3. Optional: Build your native shared library using Android NDK
4. Deploy DEX and native code, if any, to the device
5. Execute it via `app_process`

### Creating a DEX

To make a DEX from a group of JARs you need to invoke `d8` tool from Android SDK. It is located under

`<ANDROID_SDK>/build-tools/<version>`

directory, where `<version>` is one of the available build tools versions you can configure via [Android SDK Manager][sdk-manager].
It is recommended to use the latest of course especially if you use newer Java features or newer devices.

The command line you need to invoke is as follows

```bash
<ANDROID_SDK>/build-tools/<version>/d8 [--debug|--release] \
    --lib <ANDROID_SDK>/platforms/android-<platfrom-version>/android.jar \
    path/to/your_main.jar \
    path/to/runtime_dependency_1.jar \
    path/to/runtime_dependency_2.jar \
    ...
```

Where `<platfrom-version>` is one of the available Android platforms you can configure via [Android SDK Manager][sdk-manager].
Just like with build tools version using the latest and greatest is usually a good idea.

**This will produce a single file always named `classes.dex` in your current working directory**. 

No you cannot make it to produce a different name - Google command line handling is beyond moronic. You can make it put the file into a different directory via `--output` command line switch but not the file name. You want the final DEX name to be different you will have to manually rename it.

Depending on the build system you use it might or might not be tricky to collect the paths to all the runtime dependencies you might have. Here is a Gradle fragment that does so and can be dropped verbatim into you desktop Gradle script. 

```Gradle
if (project.hasProperty('android.sdk')) {

    def SDK = project.property('android.sdk')
    def BUILD_TOOLS_VER = "34.0.0"
    def ANDROID_VER = "34"

    def d8 = "${SDK}/build-tools/${ANDROID_BUILD_TOOLS_VER}/d8"
    def android_jar = "${SDK}/platforms/android-${ANDROID_VER}/android.jar"

    tasks.register('dex') {
        dependsOn("jar")
        group('build')

        inputs.files([jar.archiveFile] + configurations.runtimeClasspath.files)

        outputs.file(layout.buildDirectory.file(
                jar.archiveFileName.get().dropRight(4) + ".dex"))

        doLast {
            exec {
                workingDir layout.buildDirectory.get()
                executable d8
                args += ['--debug', '--lib', android_jar]
                args += inputs.files.collect { it.path }
            }


            ant.move file: layout.buildDirectory.file("classes.dex").get(),
                     tofile: outputs.files[0]
        }
    }
    assemble.dependsOn 'dex'
}

```

It relies on a Gradle property `android.sdk` to be set to the path of Android SDK. You can set it in `properties.gradle` or via command line as `-Pandroid.sdk=...`.

If this property is set this fragment will
- Add a new `dex` target and make `assemble` target build it
- This target will generate a `.dex` file with the same base name as your `.jar` and put it next to it.

You will need set `ANDROID_BUILD_TOOLS_VER` and `ANDROID_VER` to the values appropriate for your environment.

### Optional: Building native code under NDK

How to do this in general is out of scope for this article. If you use CMake and already have a desktop build set up then simply passing

```bash
cmake 
  -DCMAKE_TOOLCHAIN_FILE=<ANDROID_SDK>/ndk/<version>/build/cmake/android.toolchain.cmake \
  -DANDROID_ABI=x86_64 \
  -DANDROID_PLATFORM=34 \
  -DANDROID_STL=c++_static \
  ...
```

when configuring your project will make it build under NDK. As usual, replace `ANDROID_ABI` and `ANDROID_PLATFORM` with values appropriate for your environment and device.

### Deploying to the device

If you app doesn't include native code you can put the DEX file anywhere you like and can access on the device. 
If you do have native code your choice is limited to a subdirectory of `/data/local/tmp` as described in [previous article][prev-article] because the shared library needs to have execute permission.

Whichever location you choose you **should** create a clean subdirectory for your code. The reason for this will be explained in the next section.

In what follows this directory will be referred as `<DEPLOYMENT_DIR>`

### Executing it

The most convenient way to execute the DEX is using the `app_process` executable which conveniently will also make all the Android standard library classes available and pre-loaded for you. 

Here is how you run a DEX:

```bash
adb shell ANDROID_DATA=<DEPLOYMENT_DIR> app_process \
    -cp <DEPLOYMENT_DIR>/your_dex_filename.dex \
    -Djava.library.path=<DEPLOYMENT_DIR> \
    <DEPLOYMENT_DIR> \
    your.main.class.Name
```

where `your.main.class.Name` is the name of the class containing your `static void Main(String[] args)` function.

You only need the `-Djava.library.path=...` argument if you use native code.

The `ANDROID_DATA` environment variable is mandatory. `app_process` requires it to be set and will create `dalvik-cache` directory (presumably for some kind of caches) under it. This is the reason you want to do everything in a dedicated subdirectory.

Repeating `<DEPLOYMENT_DIR>` before your class name is also required though `app_process` doesn't seem to do anything with this information[^3].

If you did everything correctly the command line above should run your code and produce the output you expect.

If your code doesn't run consult logcat. It will usually have pretty good information about the reason for failure.


## References

[^1]: [Android: Compiling and running a Java command-line utility](https://billauer.co.il/blog/2022/10/android-java-dalvikvm-command-line/)
[^2]: [How to run Java programs directly on Android (without creating an APK)](https://raccoon.onyxbits.de/blog/run-java-app-android/)
[^3]: [app_process source code][app_process]
