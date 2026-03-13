---
layout: page
title: "Software"
---

Below is the list of various applications and libraries I've published over the years.

## Applications

* [wsdd-native](https://github.com/gershnik/wsdd-native){:target="_blank" rel="noopener"} 
: If you have a mix of Windows and Linux/Mac machines on your local network and use SMB file sharing 
  you might have noticed that Windows doesn't show Linux/Mac machines in its Explorer's Network view. 
  This used to work in the old times but in the more "modern" versions of Windows Microsoft changed the 
  way network discovery works. `wsdd-native` is a Unix software (works on Mac, Linux, BSD, Illumos and even Haiku)
  that fixes that.

* [Translit](https://github.com/gershnik/Translit){:target="_blank" rel="noopener"} 
: A macOS keyboard input source that allows a user familiar only with Latin alphabet keyboard to type in other 
  languages by using common Latin transliteration of the target language letters. Currently supported 
  target languages are Russian, Hebrew, Ukrainian and Belarusian.

* [Translit For Windows](https://github.com/gershnik/TranslitForWindows){:target="_blank" rel="noopener"}
: The same but for Windows.

* [Tiny Intercom](apps/intercom){:target="_blank" rel="noopener"} 
: Repurpose an old, unused iPhone/iPad/Android device or even a Mac (Apple Silicone only) into an audio intercom. Free, as in beer, with no in-app purchases or ads but not open source.

* [Keep-Awake](https://github.com/gershnik/keep-awake){:target="_blank" rel="noopener"}
: A small tool that allows you to prevent a Windows machine from sleeping/hibernating. This is useful, for example, when connecting over SSH to a Windows machine that is configured to sleep when not used.
  Unlike other solutions to this task, keep-awake doesn't change global computer settings and so doesn't leave them 'orphaned' if it is abnormally terminated.

* [wakeonlan](https://github.com/gershnik/wakeonlan){:target="_blank" rel="noopener"}
: Yet another wake-on-lan command line tool.

## Libraries

* [intrusive_shared_ptr](https://github.com/gershnik/intrusive_shared_ptr){:target="_blank" rel="noopener"} 
: An implementation of an intrusive reference counting smart pointer, highly configurable reference counted base class and adapters.

* [SimpleJNI](https://github.com/gershnik/SimpleJNI){:target="_blank" rel="noopener"} 
: A powerful lightweight C++ wrapper for JNI

* [objc-helpers](https://github.com/gershnik/objc-helpers){:target="_blank" rel="noopener"}
: An ever-growing collection of utilities to make coding on Apple platforms in C++ or ObjectiveC++ more pleasant. Some functionality is also available on Linux.

* [argum](https://github.com/gershnik/argum){:target="_blank" rel="noopener"}
: Fully-featured, powerful and simple to use C++ command line argument parser.

* [ThinSQLite++](https://github.com/gershnik/thinsqlitepp){:target="_blank" rel="noopener"}
: A thin, safe and convenient modern C++ wrapper for SQLite API.

* [SysString](https://github.com/gershnik/sys_string){:target="_blank" rel="noopener"}
: A header-only library that provides a C++ string class template optimized for interoperability with external native string types. It is immutable, Unicode-first and exposes convenient operations similar to Python or ECMAScript strings. It provides fast concatenation via `+` operator that does not allocate temporary strings. The library exposes bidirectional UTF-8/UTF-16/UTF-32 and grapheme cluster views of strings as well as of other C++ ranges of characters.

* [modern-uuid](https://github.com/gershnik/modern-uuid){:target="_blank" rel="noopener"}
: A modern, no-dependencies, portable C++ library for manipulating [UUIDs](https://en.wikipedia.org/wiki/Universally_unique_identifier), 
  [ULIDs](https://github.com/ulid/spec), [NanoIDs](https://github.com/ai/nanoid) and [Cuid2s](https://github.com/paralleldrive/cuid2).

* [processtitle](https://github.com/gershnik/processtitle){:target="_blank" rel="noopener"}
: A Python extension module allows to customize process "title" as reported by `ps`, `top`, Activity Monitor, Task Manager and similar tools.

* [PTL](https://github.com/gershnik/ptl){:target="_blank" rel="noopener"}
: A C++ wrapper library for Posix and related calls.

* [repopulator](https://github.com/gershnik/repopulator){:target="_blank" rel="noopener"}
: A portable Python library to generate binary software repositories (APT, YUM/DNF, Pacman, Alpine, and FreeBSD pkg) on any OS.


## Helpers and tools

* [android-cmdline-jni](https://github.com/gershnik/android-cmdline-jni){:target="_blank" rel="noopener"}
: Minimal code to create a native Android command-line application that loads and uses ART virtual machine.

* [libuuid-cmake](https://github.com/gershnik/libuuid-cmake){:target="_blank" rel="noopener"}
: CMake build for [libuuid](https://github.com/util-linux/util-linux/tree/master/libuuid) library from [util-linux](https://github.com/util-linux/util-linux)

* [BetterWiFiNINA](https://github.com/gershnik/BetterWiFiNINA){:target="_blank" rel="noopener"}
: A fork of [WiFiNINA](https://github.com/arduino-libraries/WiFiNINA) library that attempts to improve it and fix longstanding issues to make it usable for more serious network communication needs.

* [MbedNanoTLS](https://github.com/gershnik/MbedNanoTLS){:target="_blank" rel="noopener"}
: An updated version of Mbed TLS for Arduino Mbed OS Nano boards.

* [Multi GH Action Runner](https://github.com/gershnik/multi-gh-action-runner){:target="_blank" rel="noopener"}
: A Github Actions helper tool that orchestrates creation and running of multiple Github self-hosted runners driven by a config file. You can configure multiple runners per-repo for multiple repos and start and stop them all with a single command. The tool sets up new repos and cleans up no longer configured ones automatically at startup.

* [set-vscode-xcode](https://github.com/gershnik/set-vscode-xcode){:target="_blank" rel="noopener"}
: A script to enable VSCode native debugging on Mac with Xcode in a non-default location.
