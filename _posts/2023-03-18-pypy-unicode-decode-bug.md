---
layout: post
title:  PyPy's PyUnicode UTF-16 and UTF-32 decoders crash on NULL byteorder
tags: python c++ annoyances
---

<!-- References -->
[utf32-codecs]: <https://docs.python.org/3/c-api/unicode.html#utf-32-codecs>
[utf16-codecs]: <https://docs.python.org/3/c-api/unicode.html#utf-16-codecs>
<!-- End References -->

According to Python.org documentation for [PyUnicode_DecodeUTF16][utf16-codecs] and [PyUnicode_DecodeUTF32][utf32-codecs] you should be able to pass `nullptr` as `byteorder` parameter.

> If byteorder is NULL, the codec starts in native order mode.

This indeed works as advertised on all CPython versions I could try (>=3.7).

On PyPy however, as of this writing both version 3.7 and 3.8 (there isn't any newer one available) crash. It appears that you must pass in a valid pointer there.

If you use C++20 or higher this is actually easy.

```cpp
#include <bit>
...
int byteorder = (endian::native != endian::little) - 
                        (endian::native == endian::little);
```

If not, on GCC/CLang you can likely do 

```cpp
int byteorder = (__BYTE_ORDER__ != __ORDER_LITTLE_ENDIAN__) - 
                        (__BYTE_ORDER__ == __ORDER_LITTLE_ENDIAN__);
```

Finally on MSVC the order is always little-endian (e.g. -1 for Python) as far as I know.

For the record I did try to report this bug to PyPy team but their site requires 
registration to do so and, frankly, this is too much of a hassle.


