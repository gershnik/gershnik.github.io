---
layout: page
exclude: true
title: "Tips: C++ language"
---

{::options parse_block_html="true" /}

<div style="background-color:#f55c51;border-radius:10px;padding:10px">
<sm>This is an archive copy of old collection of tips I once wrote. People still follow links to it so I keep it alive. Note that it was written long ago and information here may be obsolete.

Use at your own risk</sm>
</div>
<br>

* TOC
{:toc}
{: #toc}

---

## Avoiding function pointers mismatch with actual function
{: #func_ptr}

To see what the problem is let's review the current practice. Often when people want to export some functions from a DLL/shared library they do it in the following way. First you create a common header like this

```cpp
//header.h

extern "C"
{										
    void [compiler specific stuff] func(int, float);
}
```

The "compiler specific stuff" above are thing like VC calling conventions `__cdecl`, `__stdcall` etc. Then you implement this function in one of your source files

```cpp
//impl.cpp

#include "header.h"

extern "C"
{
    void [compiler specific stuff] * func(int i, float f)
    {
    }
}
```

Some of your caller may want to load your DLL/shared library dynamically. They would have to include your header and use it like this (example assumes Windows so Unix programmers substitute `dlopen()` for `LoadLibrary()` and `dlsym()` for `GetProcAddress()`)

```cpp
//caller.cpp

#include "header.h"

int main()
{
    HINSTANCE dll = LoadLibrary("mydll.dll");
	
    typedef void ([compiler specific stuff] * func_ptr)(float, int);
    func_ptr f = (func_ptr)GetProcAddress(dll, "func");
	
    f(1.2f, 1);
    ... 
}

```

There is a deliberate mistake in the code above. I "accidentally" switched the order of parameters when declaring the `func_ptr` type. However, neither compiler nor linker will notice this error and most likely it will make it straight into executable code. What will happen at runtime is known as Undefined Behavior (UB). The details will depend on the exact manner of parameter passing used by your compiler. It could crash, produce wrong results or even succeed.

Note that the error I have made is just one of many possible ones. I could have used different number of parameters, use different calling convention or different return type and none of this would be detected. If you have programmed with dynamic libraries for any length of time you probably have been hit by this problem quite a few times.

One way to solve this problem is to declare the function pointer type yourself like this.

```cpp
//header.h

extern "C"
{
    void [compiler specific stuff]  func(int, float);
	
    typedef void ([compiler specific stuff] * func_ptr)(int, float);
}
```

The callers now can use the func_ptr type provided by you instead of declaring their own. This is an improvement but it still lacks elegance. Now you are responsible for synchronizing the function and pointer type signatures. And if you make a mistake you will hurt all your clients at once. Still, because this is what Microsoft does the majority of programmers use this approach.

There is an easier and more elegant way to avoid this kind of problems which is surprisingly little known. The header needs to be modified like this

```cpp
//header.h

extern "C"
{
    //not a pointer but function type
    typedef void [compiler specific stuff] func_t(int, float);
	
    //this is the function declaration
    func_t func;
	
    //and this is the pointer type just for convenience
    typedef func_t * func_ptr;
}
```

Your implementation code does not change. You callers can either use func_ptr as before or simply write

```cpp
//caller.cpp

#include "header.h"

int main()
{
    HINSTANCE dll = LoadLibrary("mydll.dll");
	
    func_t * f = (func_t *)GetProcAddress(dll, "func");
	
    f(1, 1.2f);
    ... 
}
```

Note that this solution allows you to specify function signature **once** and thus totally avoid synchronization problems.


[Back to top](#toc)

---

## Correct return of extern "C" callbacks from C++
{: #callback}

When people need to return a callback declared as `extern "C"` from C++ code they usually resort to two techniques. The first one (favored by Windows programmers) is

```cpp
//the callback type
extern "C" { typedef void (*bar_ptr)(); }

class foo { static void bar(); }

void foo::bar()
{
}

bar_ptr f = &foo::bar;
```

In other words you simply assign a C++ static function address to an `extern "C"` pointer. Though this looks elegant it is actually Undefined Behavior (UB). Standard C++ does not guarantee that an `extern "C++"` function (which what a static class method is) is compatible with `extern "C"` pointer. They could use different "calling convention", different ways to pass built-in types and in general cannot be mixed. Visual C++ and most other compilers currently does make them compatible and, for the reasons, of backward compatibility probably ever will. However, even some compilers that currently allow this warn on such usage. For example compiling this code on Sun CC will produce something like the following warning

```
Warning (Anachronism): Assigning void(*)() to extern "C" void(*)().
```

Presumably its behavior may change in future versions to make this an error.

Because of such warnings many Unix programmers resort to

```cpp
//the callback type
extern "C" { typedef void (*bar_ptr)(); }

class foo { static void bar(); }

void foo::bar()
{
}

extern "C"
void bar_forward()
{
    foo::bar();
}

bar_ptr f = &bar_forward;
```

This is correct code but it is very irritating. Every time you want to return a callback you need to declare a useless function just to make the compiler and your conscience happy. Fortunately, there is an easy workaround that is very little known

```cpp
extern "C" 
{
    //declare a function type
    typedef void bar_t();
    
    //and a pointer type for convenience
    typedef bar_t * bar_ptr;
} 

class foo { static bar_t bar; }

void foo::bar()
{
}

bar_ptr f = &foo::bar;
```

Essentially what this code does is to make a static member function of a class `extern "C"`. Make sure you read this code carefully to understand what's going on. Many people are surprised that you can do this but it is standard C++.

[Back to top](#toc)

---

## Determine whether a global/static variable is defined in a given module (exe or dll)
{: #global_exe_dll}

This is obviously platform specific and non-standard (the C++ standard is still ignorant of DLLs even though they have been around for a long time). So to be more specific how can a C or C++ code on Windows determine whether a given global or static variable is defined in a given module.

A module on Windows is either a main executable or any of DLLs it loads at runtime. A module is identified by module handle which has type `HMODULE`. Our goal is to write the following function

```cpp
bool IsVariableInModule(void * pVar, HMODULE hMod)
```

where `pVar` is a pointer to the global/static variable and `hMod` is the module handle

The task is incredibly easy. All modules occupy a contiguous region in memory and any global/static variables defined in them will reside somewhere in this region. The beginning of this region is nothing else but `HMODULE` "handle" itself. That's right, unlike other Windows handles this one is simply a user mode pointer to the beginning of the module. To find the end all we need to do is to peek into the module's portable executable (PE) headers. PE is simply the binary format of Windows executables (exes and dlls) and it preserved when a module is loaded into memory from file. More information about PE format can be found here. It turns out that one of the fields of PE header gives exactly what we need: the size of the module in bytes. Putting it all together we get this simple function

```cpp
bool IsVariableInModule(void * pVar, HMODULE hMod)
{
    const IMAGE_DOS_HEADER * const pDOSHeader = (const IMAGE_DOS_HEADER *)hMod;
    const IMAGE_NT_HEADERS * const pNTHeader = 
        (const IMAGE_NT_HEADERS *)((BYTE*)pDOSHeader + pDOSHeader->e_lfanew);
    const size_t ImageSize = pNTHeader->OptionalHeader.SizeOfImage;

    return (p >= pDOSHeader && p <= (BYTE*)pDOSHeader + ImageSize);

}
```
That's it. 🙂


[Back to top](#toc)

---

## How to printf() into std::string
{: string_printf}

If you work on any real-life project, sooner or later you may hit this problem. How to perform `printf()`-style output with the destination being `std::string`? With C++11 this is perfectly possible:

```cpp
int sprintf(std::string & res, const char * format, ...)
{
    va_list vl;
    va_start(vl, format);
    const int size = vsnprintf(0, 0, format, vl);
    va_end(vl);
    if (size <= 0)
        return size;
    size_t appendPos = res.size();
    size_t appendSize = size_t(size) + 1;
    res.resize(appendPos + appendSize);
    va_start(vl, format);
    const int ret = vsnprintf(&res[appendPos], appendSize, format, vl);
    va_end(vl);
    res.resize(appendPos + size_t(ret > 0 ? ret : 0));
    return ret;
}
```

This appends to the existing string. It should be trivial to change if an "overwrite" semantics is desired.

Note the re-initialization of `va_list`. On some platforms this might not be necessary but on others the first call to `vsnprintf` will
"consume" it making the second call access random memory.

[Back to top](#toc)

---

## atexit() and dynamic/shared libraries
{: #atexit_dll}

C and C++ standard libraries include a sometimes useful function: `atexit()`. It allows the caller to register a callback that is going to be called when the application exits (normally). In C++ it is also integrated with the mechanism that calls destructors of global objects so things that were created before a given call to `atexit()` will be destroyed after the callback and vice versa. All this should be well known and it works perfectly fine until DLLs or shared libraries enter the picture.

The problem is, of course, that dynamic libraries have their own lifetime that, in general, could end before the main application's one. If a code in a DLL registers one of its own functions as an `atexit()` callback this callback should better be called before the DLL is unloaded. Otherwise, a crash or something worse will happen during the main application exit. (To make things nasty crashes during exit are notoriously hard to debug since many debuggers have problem dealing with dying processes).

This problem is much better known in the context of the destructors of C++ global objects (which, as mentioned above, are `atexit()`'s brothers). Obviously any C++ implementation on a platform that supports dynamic libraries had to deal with this issue and the unanimous solution was to call the global destructors either when the shared library is unloaded or on application exit, whichever comes first.

So far so good, except that some implementations "forgot" to extend the same mechanism to the plain old `atexit()`. Since C++ standard doesn't say anything about dynamic libraries such implementations are technically "correct", but this doesn't help the poor programmer who for one reason or another needs to call `atexit()` passing a callback that resides in a DLL.

On the platforms I know about the situation is as follows. MSVC on Windows, GCC on Linux and Solaris and SunPro on Solaris all have a "right" `atexit()` that works the same way as global destructors. However, GCC on FreeBSD at the time of this writing has a "broken" one which always registers callbacks to be executed on the application rather than shared library exit. However, as promised, the global destructors work fine even on FreeBSD.

What should you do in portable code? One solution is, of course, to avoid `atexit()` completely. If you need its functionality it is easy to replace it with C++ destructors in the following way

```cpp
//Code with atexit()

void callback()
{
    //do something
}

...
atexit(callback);
...

//Equivalent code without atexit()

class callback
{
public: 
    ~callback()
    {
        //do something
    }
    
    static void register();
private:
    callback()
    {}
    
    //not implemented
    callback(const callback &);
    void operator=(const callback &); 
};

void callback::register()
{
    static callback the_instance;
}

...
callback::register();
...
```

This works at the expense of much typing and non-intuitive interface. It is important to note that there is no loss of functionality compared to `atexit()` version. The callback destructor cannot throw exceptions but so do functions invoked by `atexit()`. The `callback::register()` function may be not thread safe on a given platform but so is `atexit()` (C++ standard is currently silent on threads so whether to implement `atexit()` in a thread-safe manner is up to implementation)

What if you want to avoid all the typing above? There usually is a way and it relies on a simple trick. Instead of calling broken `atexit()` we need to do whatever the C++ compiler does to register global destructors. With GCC and other compilers that implements so-called Itanium ABI (widely used for non Itanium platforms) the magic incantation is called `__cxa_atexit`. Here is how to use it. First put the code below in some utility header

```cpp
#if defined(_WIN32) || defined(LINUX) || defined(SOLARIS)

	#include <stdlib.h>

	#define SAFE_ATEXIT_ARG 
	
	inline void safe_atexit(void (*p)(SAFE_ATEXIT_ARG)) 
	{
	    atexit(p);
	}

#elif defined(FREEBSD)

	extern "C" int __cxa_atexit(void (*func) (void *), void * arg, void * dso_handle);
	extern "C" void * __dso_handle;		


	#define SAFE_ATEXIT_ARG void *
	
	inline void safe_atexit(void (*p)(SAFE_ATEXIT_ARG))
	{
	    __cxa_atexit(p, 0, __dso_handle);
	}

#endif
```

And then use it as follows

```cpp
void callback(SAFE_ATEXIT_ARG)
{
    //do something
}

...
safe_atexit(callback);
...
```

The way `__cxa_atexit` works is as follows. It registers the callback in a single global list the same way non-DLL aware `atexit()` does. However it also associates the other two parameters with it. The second parameter is just a nice to have thing. It allows the callback to be passed some context (like some object's this) and so a single callback can be reused for multiple cleanups. The third parameter is the one we really need. It is simply a "cookie" that identifies the shared library that should be associated with the callback. When any shared library is unloaded its cleanup code traverses the `atexit` callback list and calls (and removes) any callbacks that have a cookie that matches the one associated with the library being unloaded. What should be the value of the cookie? It is not the DLL start address and not its `dlopen()` handle as one might assume. Instead the handle is stored in a special global variable `__dso_handle` maintained by C++ runtime.

The `safe_atexit` function **must** be inline. This way it picks whatever `__dso_handle` is used by the calling module which is exactly what we need.

Should you use this approach instead of the verbose and more portable one above? Probably not, though who knows what requirements you might have. Still, even if you don't ever use it, it helps to be aware of how things work so this is why it is included here.

[Back to top](#toc)

