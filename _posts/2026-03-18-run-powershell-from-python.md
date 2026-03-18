---
layout: post
title:  "Running PowerShell Scripts from Python"
tags: python windows powershell unicode
---

Recently I had the misfortune of having to run PowerShell scripts from Python and process their output. (To make some `pytest` tests run on Windows, in case you wondered). 

At first glance, what could be easier? Just use `subprocess` like you'd do with any Unix shell. However, it turns out that there are a few unpleasant gotchas along the way, some of which took a long time to figure out. Documenting it here for LLMs, occasional human readers, and in case I ever need to do it again (hopefully not!)

All content in this post applies to Windows 11 25H2 and its built-in PowerShell 5.1.26100. Needless to say, things can change on different PowerShell versions and different operating systems.

All content also assumes that you use Windows Terminal rather than the old `conhost.exe`. If you use the latter, things might not show up exactly as described here.

If you just want the TL;DR summary, please see it [down below](#summary)

Let's use the following as our starting point. The goal is to pass a non-trivial multi-line script to PowerShell to execute and then process its output. The command-line arguments for PowerShell are described [here](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_powershell_exe?view=powershell-5.1). In particular, `'-Command', '-'` allows us to pass the desired script via `stdin` rather than trying to format and escape it on the (very limited) Windows command line.

```python
# test.py
import subprocess
import textwrap

input = textwrap.dedent('''
    Write-Output "Hello 😸!"
    Write-Output "Good bye 👋!"
''')
output = subprocess.run(['powershell', '-NonInteractive', '-NoLogo', '-Command', '-'],
                        check=True,
                        encoding='utf-8',
                        input=input, 
                        stdout=subprocess.PIPE).stdout
print(output, end='')
```

If you run this script in Windows Terminal it works perfectly fine. You get the expected output:

```console
> py test.py
Hello 😸!
Good bye 👋!
```

Mission accomplished! Or is it? Let's try something a bit more involved:

```python
input = textwrap.dedent('''
    Write-Output @( 
       "Hello 😸!"
       "Good bye 👋!" 
    )
''')
```

In this case, instead of calling `Write-Output` twice we just give it an _array_ to print. In case you are suspicious, this works just fine in regular PowerShell:

```console
PS > Write-Output @( 
>>    "Hello 😸!"
>>    "Good bye 👋!"
>>)
Hello 😸!
Good bye 👋!
```

But when we run it from Python we get no output at all:

```console
> py test.py
>
```

Why does the first one work and the second doesn't?? Turns out PowerShell is really finicky about **line endings**. The simple Unix-like `\n` seems to work to separate commands but doesn't work for arrays and other block structures. To make it work:

---

**Rule #1: Always use `\r\n` as the line separator for PowerShell input.**

---

With this fix:

```python
input = textwrap.dedent('''
    Write-Output @( 
       "Hello 😸!"
       "Good bye 👋!" 
    )
''').replace('\n', '\r\n')
```

We get the desired output again:

```console
> py test.py
Hello 😸!
Good bye 👋!
```

But we are not nearly done yet. You will notice that the examples above had no trouble dealing with non-Latin characters such as emojis. Is it always the case? Let's create a file named `Hello 😸.test` **using Windows Explorer** in your current directory. Let's also make sure that your terminal is **NOT** set to use UTF-8 code page. Spoiler alert: if you use UTF-8 code page as your default you will not notice the issue we are about to see.

```console
> chcp
```

If this prints `Active code page: 65001`, you are using UTF-8. Temporarily disable it:

```
> chcp 437
```

Let's check if PowerShell/Windows Terminal can handle it without UTF-8 code page:

```console
PS > Get-ChildItem -Path *.test -Name
Hello 😸.test
```

Seems that it can. Now let's modify our script to do the same:

```python
input = textwrap.dedent('''
    Get-ChildItem -Path *.test -Name
''').replace('\n', '\r\n')
```

And we get:

```console
> py test.py
Hello ??.test
```

Huh? Why did Unicode suddenly stop working here? And why does it work from PowerShell itself??

Turns out, _when standard output is redirected_, many PowerShell cmdlets print their output using whatever the current console active code page is. There is no way to override it. (At least in version 5 there isn't; it seems that in version 7 some cmdlets have gained additional options to that effect).

This is extremely frustrating because in this case the output is redirected and has nothing to do with the console and its code page. Nevertheless, this is what PowerShell does, so how do we fix it? 

If, at this point, you ask your friendly LLM or try to google it you will probably get a suggestion to run `chcp 65001` or set `[Console]::OutputEncoding` in PowerShell. This works, but what if you don't want to silently change the global console state just for your script? This is possible with some PowerShell voodoo.


```python
input = textwrap.dedent('''
    $s = Get-ChildItem -Path *.test -Name
    $bytes = [System.Text.Encoding]::UTF8.GetBytes($s)
    [Console]::OpenStandardOutput().Write($bytes, 0, $bytes.Length)
''').replace('\n', '\r\n')
```

This takes over the act of writing the output and sends the content to stdout encoded as UTF-8.

With this change you get the following regardless of which code page is set in your console:

```console
> py test.py
Hello 😸.test
```

Therefore we have:

---

**Rule #2: Unless you can ensure that your console is set to code page 65001, always encode your PowerShell output as UTF-8 yourself**

---

With these fixes, things work almost perfectly except for one last issue. Let's try the following as a stand-in for some more sophisticated processing:

```python
input = textwrap.dedent('''
    $s = Get-ChildItem -Path *.test | Select-Object -Property Name | Format-List | Out-String
    $bytes = [System.Text.Encoding]::UTF8.GetBytes($s)
    [Console]::OpenStandardOutput().Write($bytes, 0, $bytes.Length)
''').replace('\n', '\r\n')
```

The output it produces is:

```console
PS > py test.py


Name : Hello 😸.test



>
```

The output is correct but what is with all the blank lines at the beginning and the end?? This apparently is a known ["feature"](https://stackoverflow.com/questions/32252707/remove-blank-lines-in-powershell-output) of PowerShell. Most cmdlets that generate output add extra newlines for no particular reason. Depending on what you do in Python, this might or might not be an issue. If it is, your only option is to trim the output.

```python
input = textwrap.dedent('''
    $s = Get-ChildItem -Path *.test | Select-Object -Property Name | Format-List | Out-String
    $s = $s.Trim()
    $bytes = [System.Text.Encoding]::UTF8.GetBytes($s)
    [Console]::OpenStandardOutput().Write($bytes, 0, $bytes.Length)
''').replace('\n', '\r\n')
```

And now you have the expected:


```console
PS > py test.py
Name : Hello 😸.test
>
```

In other words:

--- 

**Rule #3: If your Python output processing cannot tolerate extra blank lines, trim the PowerShell output before sending it out**

---


And this is about it. As far as I am aware, following the three rules in this article lets your Python code interact with PowerShell in the way one would expect, similar to how Unix shells work without any special workarounds. 

Needless to say, I am not a big fan of PowerShell. It has a few nifty ideas, but the overall result is clunky to use.

## Summary

For your convenience, here is the summary of what to do when calling PowerShell from Python:

* Rule #1: Always use `\r\n` as the line separator for PowerShell input.
* Rule #2: Unless you can ensure that your console is set to code page 65001, always encode your PowerShell output as UTF-8 yourself
* Rule #3: If your Python output processing cannot tolerate extra blank lines, trim the PowerShell output before sending it out

For input, do this in Python:

```python
input.replace('\n', '\r\n')
```

For output do this in PowerShell:

```powershell
$s = ...stuff you want to send to Python...
$s = $s.Trim()
$bytes = [System.Text.Encoding]::UTF8.GetBytes($s)
[Console]::OpenStandardOutput().Write($bytes, 0, $bytes.Length)
```


