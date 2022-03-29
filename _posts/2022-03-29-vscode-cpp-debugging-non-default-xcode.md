---
layout: post
title:  "VSCode: native debugging with Xcode in a non-default location"
tags: c++ vscode macos xcode
---

* TOC
{:toc}

<!-- References -->
[ms-info]: <https://code.visualstudio.com/docs/cpp/lldb-mi#_using-an-lldbframework-not-installed-via-xcode>
[script]: <https://github.com/gershnik/set-vscode-xcode>
<!-- End References -->

## Problem

If you are like me you use a Mac with often multiple versions of Xcode on it, installed into a 
non-default (e.g. not `/Applications/Xcode`) location. You also like to use VSCode to work on 
cross-platform native projects. 

If you do both of these things you have undoubtedly discovered that VSCode is unable to debug. 
For reasons known only to its C++ extension developers it can only use `lldb` debugger if `LLDB.framework` is found in one of its default locations. 

The [official documentation][ms-info] hints at a possible workaround:

> 1. Copy the `lldb-mi` executable in `~/.vscode/extensions/ms-vscode.cpptools-<version>/debugAdapters/lldb-mi/bin` to the folder where the `LLDB.framework` is located.
> 2. Add the full path of `lldb-mi` to `miDebuggerPath` in your `launch.json` configuration.

but this doesn't make much sense when `LLDB.framework` comes from Xcode. You don't want to copy anything into your Xcode installation directory! You probably don't want to have to type a path deep inside Xcode bundle into your `launch.json` either.

More problematically, even if you did all that, it still doesn't work when using CMake extension
and debugging from it. This doesn't use `launch.json` and is blissfully unaware of your custom `lldb-mi`.


## Solution

Fortunately, there is another way and it can even be made relatively painless with some scripting.
The general approach is:

1. Create a **symbolic link** to desired Xcode's `LLDB.framework` in `~/.vscode/extensions/ms-vscode.cpptools-<version>/debugAdapters/lldb-mi/bin`. Note that this is a user directory and so ok to write to. This alone takes care of CMake debugging.
2. Create a **symbolic link** somewhere easy to remember to `~/.vscode/extensions/ms-vscode.cpptools-<version>/debugAdapters/lldb-mi/bin/lldb-mi`. Then use that link as `miDebuggerPath` setting. This takes care of `launch.json`-style debugging.
  \
  \
  For example you could do something like 
  ```bash
mkdir ~/.vscode-lldb-mi
ln -s ~/.vscode/extensions/ms-vscode.cpptools-1.9.7/debugAdapters/lldb-mi/bin/lldb-mi ~/.vscode-lldb-mi/lldb-mi
  ```
  And then put the following in `launch.json` 
  ```json
{
    "name": "(lldb) Launch",
    "type": "cppdbg",
    "request": "launch",
    "program": "...path to my program...",
    "args": [],
    "stopAtEntry": false,
    "cwd": "...dir of my program...",
    "environment": [],
    "externalConsole": false,
    "osx": {
        "MIMode": "lldb",
        "miDebuggerPath": "${env:HOME}/.vscode-lldb-mi/lldb-mi"
    }
}
  ```

This works but requires tedious manual maintenance. Every time you want to switch to a different Xcode you need to manually update the symbolic link. More annoyingly, every time C++ extension 
is updated (which is often) it creates a new folder for itself under ~/.vscode/extensions` and you need to re-create the symbolic link again. 

To avoid the manual hassle I've created a script that automates the symbolic link creation. It can be found [in this Github repo][script]. The `README` file there describes the usage which, hopefully, should be straightforward.

Of course, as with any workarounds the information here is accurate at the moment it was last updated. If you read this significantly later perhaps VSCode team have already addressed the problem (one can hope!) or the steps and the script might not work anymore.





