---
layout: post
title:  "Efficient reverse UTF-8 decoder"
date:   2021-03-24 07:08:00 -0700
tags: c++ unicode
---

Wait, what is a **reverse** UTF-8 decoder and what do you need it for? 

Consider a UTF-8 encoded Unicode string that you want to process from back to front for some reason. Perhaps you want to remove trailing whitespace (like `rtrim` function in Python does) or locate last instance of some Unicode character based on some criteria. 

If what you are looking for is a simple character or a string you can get away by just reverse searching for its UTF-8 represenation as-is. UTF-8 is specifically designed to be allow it - no character forms a part of another (like some ancient encodings used to do). However, if you criteria more complex - perhaps you are looking for Unicode character properties - this approach is no longer feasible. 

Of course you can just convert the entire string to UTF-32 first and search backwards in it but this is quite inelegant and wasteful. Shouldn't it be possible to go backward in the string and decode each UTF-32 codepoint in turn?

It is indeed possible. The way [UTF-8](https://en.wikipedia.org/wiki/UTF-8) is specified allows to unambigously decode a character by looking at its encoding form in reverse order. However, at the time of this writing I wasn't able to find any existing algorithm to do so.

The best known forward UTF-8 decoder (as far as I am aware) is [Björn Höhrmann's](https://bjoern.hoehrmann.de/utf-8/decoder/dfa/) one. It uses a clever state machine technique to avoid a long sequence of performance killing `if` statements that slow down naive brute force decoders. 

It is possible to apply the same principles to reverse decoding. If you are anxious for the code you can jump directly to the [Code](#code) section. If you want to know how it works, read on.

## State machine

The state machine for reverse decoding is as follows. Just like with the forward decoding we start in state zero, and whenever we come back to it, we've seen a whole Unicode character. Transitions not in the graph are disallowed; they all lead to state one - the error state.

![State machine](/images/utf8-decoder-state-machine.svg)

We will use the same character classes as Höhrmann's decoder - they have a nice property of removing the topmost bits of each byte and allow easier computation of codepoint value. With the character classes the state machine looks like this:

![State machine with char classes](/images/utf8-decoder-state-machine-classes.svg)

With this state machine implementation is more or less straighforward. For each character we need to lookup it's class then compute the next state based on the state machine while reconstructing the UTF-32 codepoint.

## Code

The code for the decoder is given below. It has two notable differences from Höhrmann's

* It is written in C++ (C++17 to be precise). If you want it in C or any other language, the transformation should be pretty straightforward
* It does not get "stuck" in error state. When a new character arrives in the error state the decoding starts again just like from 'accepted' state. I found this semantics more convenient than having sticky errors and resetting the decoder manually.

Similar to Höhrmann's the state values are pre-multiplied by 12 to enable convenient acces to them in the state table.

```cpp
//
// Copyright 2020 Eugene Gershnik
//
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file or at
// https://github.com/gershnik/gershnik.github.io/blob/main/LICENSE
//
class reverse_utf8_decoder
{
public:
    constexpr void put(uint8_t byte) noexcept
    {
        uint32_t type = s_state_table[byte];
        m_state = s_state_table[256 + m_state + type];

        if (m_state <= state_reject)
        {
            m_collect |= (((0xffu >> type) & (byte)) << m_shift);
            m_value = char32_t(m_collect);
            m_shift = 0;
            m_collect = 0;
        }
        else
        {
            m_collect |= ((byte & 0x3fu) << m_shift);
            m_shift += 6;
        }
    }
    
    constexpr bool done() const noexcept
        { return m_state == state_accept; }
    
    constexpr bool error() const noexcept
        { return m_state == state_reject; }
    
    constexpr char32_t value() const noexcept
        { return m_value; }
private:
    char32_t m_value;
    uint32_t m_collect = 0;
    uint8_t m_shift = 0;
    uint8_t m_state = state_accept;
    
    static constexpr uint8_t state_accept = 0;
    static constexpr uint8_t state_reject = 12;
    
    static constexpr const uint8_t s_state_table[] = {
    // The first part of the table maps bytes to character classes that
    // to reduce the size of the transition table and create bitmasks.
         0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,  0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
         0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,  0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
         0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,  0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
         0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,  0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,
         1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,  9,9,9,9,9,9,9,9,9,9,9,9,9,9,9,9,
         7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,  7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,7,
         8,8,2,2,2,2,2,2,2,2,2,2,2,2,2,2,  2,2,2,2,2,2,2,2,2,2,2,2,2,2,2,2,
        10,3,3,3,3,3,3,3,3,3,3,3,3,4,3,3, 11,6,6,6,5,8,8,8,8,8,8,8,8,8,8,8,
        
    // The second part is a transition table that maps a combination
    // of a state of the automaton and a character class to a state.
    //   0  1  2  3  4  5  6  7  8  9 10 11
         0,24,12,12,12,12,12,24,12,24,12,12,
         0,24,12,12,12,12,12,24,12,24,12,12, 
        12,36, 0,12,12,12,12,48,12,36,12,12,
        12,60,12, 0, 0,12,12,72,12,72,12,12,
        12,60,12, 0,12,12,12,72,12,72, 0,12,
        12,12,12,12,12, 0, 0,12,12,12,12,12,
        12,12,12,12,12,12,12,12,12,12,12, 0
    };
};
```

That's it. With this decoder you can iterate over UTF-8 backwards. For example

```cpp

template<class InIt, class Pred>
std::pair<InIt, InIt> find_reverse_utf8(InIt first, InIt last, Pred pred)
{
    reverse_utf8_decoder decoder;
    InIt start = first;
    while(first != last)
    {
        decoder.put(*first);
        ++first;
        if (decoder.done())
        {
            if (pred(decoder.value()))
              return {start, first};
            start = first;
        }
        else if (decoder.error())
        {
            throw std::runtime_error("invalid UTF-8");
        }
    }
    if (!decoder.done())
       throw std::runtime_error("invalid UTF-8");
    return {last, last};
}

//Let's find the last Unicode whitespace

std::string str = "a bc\u205Fxyz";

auto [rstart, rend] = find_reverse_utf8(str.rbegin(), str.rend(), [](char32_t c) {
    constexpr const char32_t whitespaces[] =
        U"\u0009\u000A\u000B\u000C\u000D\u0020\u0085\u00A0"
        U"\u1680\u2000\u2001\u2002\u2003\u2004\u2005\u2006"
        U"\u2007\u2008\u2009\u200A\u2028\u2029\u202F\u205F\u3000";

    return std::char_traits<char32_t>::find(whitespaces, std::size(whitespaces) - 1, c) != nullptr;;
});

assert(rend.base() == str.begin() + 4);

```

## A note on error handling

In the example above the code threw an exception on invalid UTF-8. Often you want to handle it gracefully, substituting `U'\uFFFD'` for invalid character like UTF-8 decoders commonly do. In this case forward decoders usually use the following heuristic: restart from the last character that cause the error unless it was the first character in a codepoint. The idea here is that if the first character in a codepoint is bad we just move to the next one. If any of the trail chactaers are wrong we assume that everything before is one bad character and restart from there. (It is possible to have more sophisticated handling but it slows things down).

For reverse decoding this approach works but produces suboptimal results (as in "very different from what forward decoder would produce"). Since we read backwards an error almost always indicates that everything we read so far is unrecoverable garbage and needs to be replaced. Thus, it is better to simply always restart from the next character. For example

```cpp

template<class InIt, class Pred>
std::pair<InIt, InIt> find_reverse_utf8(InIt first, InIt last, Pred pred)
{
    reverse_utf8_decoder decoder;
    InIt start = first;
    while(first != last)
    {
        decoder.put(*first);
        ++first;
        if (decoder.done())
        {
            if (pred(decoder.value()))
              return {start, first};
            start = first;
        }
        else if (decoder.error())
        {
            if (pred(U'\uFFFD'))
              return {start, first};
            start = first;
        }
    }
    if (!decoder.done())
    {
        if (pred(U'\uFFFD'))
          return {start, last};
    }
    return {last, last};
}


```  

