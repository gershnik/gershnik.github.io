---
layout: page
exclude: true
title: "Tips: Visual C++"
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

[aaad]: https://web.archive.org/web/20101210145713/http://support.microsoft.com:80/kb/555290
[baau]: https://source.winehq.org/git/wine.git/blob_plain/HEAD:/dlls/msvcrt/undname.c
[craa]: https://learn.microsoft.com/en-us/windows/win32/api/dbghelp/nf-dbghelp-undecoratesymbolname

---

## URL shortcuts in VC projects
{: #url_shortcut}

If you try to add a URL shortcut, that is a file with .url extension into VC 7+ project you will discover that the "Add" dialog does not allow it. The workaround is to drag and drop the .url file from Windows Explorer to the project in IDE.

This [knowledge base article][aaad] I authored provides more details about this issue.


[Back to top](#toc)

---

## How to "demangle" a mangled C++ name
{: #undname}

This question pops up quite often on the Internet. Usually people need to demangle names for debugging or diagnostics purposes. Since C++ standard doesn't specify a name mangling scheme (or even forces compilers to use it -- mangling is just an artifact of using C-compatible linkers and loaders) this topic is necessarily compiler-dependent. With VC you have 3 options of doing it.

1. Use `undname.exe` command line utility. It is supplied with VC6 and above and is documented in MSDN. This method works reliably but is obviously inconvenient if you need to undecorate names in your own code. The utility is located in `Common\Tools` directory for VC6 and under `VC\bin` directory with later versions.
2. Use [UnDecorateSymbolName][craa] API. This is a documented official API but unfortunately it works only for function names (not classes). If you only need to demangle function names this is the best approach.
3. Use undocumented function `_unDName`. This is the internal function used by `type_info::name()` and it is capable of undecorating any C++ name. Below is its prototype and sample usage (note that all of this is a result of reverse engineering and I have no idea about real names of flags etc.)

```cpp
#define UNDNAME_COMPLETE                 (0x0000)
#define UNDNAME_NO_LEADING_UNDERSCORES   (0x0001) /* Don't show __ in calling convention */
#define UNDNAME_NO_MS_KEYWORDS           (0x0002) /* Don't show calling convention at all */
#define UNDNAME_NO_FUNCTION_RETURNS      (0x0004) /* Don't show function/method return value */
#define UNDNAME_NO_ALLOCATION_MODEL      (0x0008)
#define UNDNAME_NO_ALLOCATION_LANGUAGE   (0x0010)
#define UNDNAME_NO_MS_THISTYPE           (0x0020)
#define UNDNAME_NO_CV_THISTYPE           (0x0040)
#define UNDNAME_NO_THISTYPE              (0x0060)
#define UNDNAME_NO_ACCESS_SPECIFIERS     (0x0080) /* Don't show access specifier (public/protected/private) */
#define UNDNAME_NO_THROW_SIGNATURES      (0x0100)
#define UNDNAME_NO_MEMBER_TYPE           (0x0200) /* Don't show static/virtual specifier */
#define UNDNAME_NO_RETURN_UDT_MODEL      (0x0400)
#define UNDNAME_32_BIT_DECODE            (0x0800)
#define UNDNAME_NAME_ONLY                (0x1000) /* Only report the variable/method name */
#define UNDNAME_NO_ARGUMENTS             (0x2000) /* Don't show method arguments */
#define UNDNAME_NO_SPECIAL_SYMS          (0x4000)
#define UNDNAME_NO_COMPLEX_TYPE          (0x8000)

extern "C"
char * _unDName(
    char * outputString,
    const char * name,
    int maxStringLength,
    void * (* pAlloc )(size_t),
    void (* pFree )(void *),
    unsigned short disableFlags);
    
const char * const pName = <decorated name>
char * const pTmpUndName = _unDName(0, pName, 0, malloc, free, UNDNAME_NO_ARGUMENTS | UNDNAME_32_BIT_DECODE);
if (pTmpUndName)
{
    //copy pTmpUndName somewhere, this is the undecorated name
    free(pTmpUndName);
}
```

If your linker cannot find `_unDName` it is exported from VC CRT dll (`msvcrt.dll`, `msvcr71.dll` etc. —— whatever the name is on your version). The `disableFlags` arguments is the same as `Flags` argument to `UnDecorateSymbolName` but not all manifest constants defined in the code above are documented in MSDN. Those that aren't are taken from WINE project [sources][baau] which helpfully reverse-engineered VC runtime.



[Back to top](#toc)

---

## How to write while(true) without warnings
{: #endless_loop}

In VC 7 and above if you try to write a loop like

```cpp
while(true)
{
    ...
}
```
you will receive a warning (on warning level 4)

```
warning C4127: conditional expression is constant
```

Which tries to protect us from accidentally mistyping the loop condition. What to do if you really want the condition to always be true and exit the loop in some other way? Obviously, you can disable the warning but doing so removes the protection it gives. A better solution is to write the loop in the following way

```cpp
for( ; ; )
{
    ...
}
```

It has the same meaning but doesn't produce the warning. Obviously MS assumes that nobody can write an empty for loop by mistake.