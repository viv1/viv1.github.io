---
layout: post
title: Are Lua strings actually just bytes ?
date: 2024-04-28 15:05
category: Programming
tags: ["lua", "string", "byte", "programming", "encording", "unicode", "utf-8"]
description: Here we explore Lua's uncommon approach to handling strings and bytes as a single entity, and its advantages, challenges, and practical considerations
---

<!-- TOC start (generated with https://github.com/derlin/bitdowntoc) -->

- [Introduction](#introduction)
- [Taking a step back - Recalling bytes and strings](#taking-a-step-back-recalling-bytes-and-strings)
- [Treatment of string in Lua](#treatment-of-string-in-lua)
   * [What makes it great](#what-makes-it-great)
   * [What makes it problematic](#what-makes-it-problematic)
- [Interconversion: Bytes to Strings and Vice Versa](#interconversion-bytes-to-strings-and-vice-versa)
- [Handling Multi-byte Encoded Characters](#handling-multi-byte-encoded-characters)
- [Conclusion](#conclusion)
- [References](#references)
- [Appendix](#appendix)
   * [Brief on UTF-8 Encoding Basics:](#brief-on-utf-8-encoding-basics)

<!-- TOC end -->

<!-- TOC --><a href='#' name="introduction"></a>
## Introduction

Most programming languages have clear distinction between strings and bytes. Lua on the other hand treats bytes and strings as same. TThis approach offers simplicity and efficiency but also presents challenges, particularly when dealing with characters beyond the basic `ASCII` set. We are going to delve into this specific behaviour of lua.

<!-- TOC --><a href='#' name="taking-a-step-back-recalling-bytes-and-strings"></a>
## Taking a step back - Recalling bytes and strings

> TL;DR version is that `bytes` are how machines store the `strings`, and `strings` are human  readable characters.
{:.prompt-info}

In the world of computer storage and processing, everything is 0 or 1. This is what we call a `bit`. In most modern computers, when we combine `8 bits`, it forms what we call a `byte`. Mathematically, we can store `2^8 (or 256)` values of in a `single byte`. 

In any programming language, when we define a character in a variable, say `myVariable = "a"`, then this is actually stored in binary format. What is the value of that binary format depends on the *encoding scheme* we decide to use to represent this character (say, `ascii`, `utf-8`, `utf-16`, etc). In `ascii`, this will be stored as `01100001` (`1 byte`). In `utf-16`, it is stored as `00000000 01100001`

> Find more details on utf-8 encoding in the [Appendix](#appendix).
{:.prompt-info}

<!-- TOC --><a href='#' name="treatment-of-string-in-lua"></a>
## Treatment of string in Lua

Lua treats each string as a sequence of bytes, and not a sequence of characters. This conflation of strings and bytes means that operations on strings often directly manipulate byte data. 

<!-- TOC --><a href='#' name="what-makes-it-great"></a>
### What makes it great

As we can imagine, there are certain advantages with this:

- **Simplicity**: Lua's approach simplifies string manipulation, as we don't need to worry about separate data types or complex conversions or do character encodings.

```lua
local greeting = "Hello"
local spaceByte = string.char(32)  -- ASCII value for space (' ')
local name = "World"

-- Concatenate strings and bytes
local message = greeting .. spaceByte .. name

print(message)  -- Output: Hello World

```

- **Efficiency**: `Byte-based` operations are often faster, especially when working with binary data or low-level network protocols.

- **Flexibility**: Lua allows us to directly access and manipulate individual bytes within a string, providing fine-grained control. This is specially advantageous when dealing with file IO or network communications. We can do direct manipulation of any data received over the network or from the file, and not worry about the character encoding used.

```
local function modifyNetworkData(bytedata)
    local dataLength = #bytedata --length of incoming bytes
    return dataLength .. bytedata
end

```

<!-- TOC --><a href='#' name="what-makes-it-problematic"></a>
### What makes it problematic

However, this approach also presents challenges, particularly when dealing with characters beyond the basic `ASCII` set:

- **Counter-intuitive outputs**: Simple operations like `string.len` can often yield counter-intuitive results.

```lua
> print(string.len('hello'))
5
> print(string.len('héllo'))
6
```
Notice this ? This is because `string.len` actually returns number of bytes and not number of characters.

`é` is actually formed using 2 bytes:
```lua
> string.len(`é`)
2
```

- **Data Integrity**: Lua string characters are each treated as `single byte`. However, in the world of `unicode`, this is not always true. Sometimes, a character uses 2, 3 or even 4 `bytes`. (See [Appendex](#appendix). Accurate processing requires recognition of entire byte sequences. Incorrect byte manipulation can lead to data corruption, particularly with international text. So, multi byte characters have to be handled properly.

For instance, 
'😀' uses 4 bytes (`#x -> 4`, In hex: `0xF09F9880`). Note `Ox` represents hexadecimal representation. We can try to get each byte representation using:

```lua
x = '😀'
> string.byte(x, 1)
240 -- OxFO -> ɀ
> string.byte(x, 2)
159 -- Ox9F -> ř
> string.byte(x, 3)
152 -- OxF9 -> Œ
> string.byte(x, 4)
128 -- 0x80 -> Ĩ
```

The individual bytes represent a completely different characters than the original, i.e.`ɀ`, `ř`, `Œ`, and `Ĩ` in order. This is because `😀` is not just a combination of 4 `single bytes`. It's actually a a more complicated representation. You can read more on it in the [Appendix Section](#appendix).

Understanding this helps in recognizing why data corruption or errors in text rendering occur if a UTF-8 sequence is improperly split or if individual bytes are incorrectly interpreted as complete characters. In practice, handling text data with care ensures that sequences are kept intact to avoid encoding errors or misinterpretation of the data. This is crucial for programming, data transmission, and storage, where even a single byte’s misplacement can lead to unexpected results or failures in processing multilingual text.


<!-- TOC --><a href='#' name="interconversion-bytes-to-strings-and-vice-versa"></a>
## Interconversion: Bytes to Strings and Vice Versa

Lua provides functions for converting between bytes and strings:
- `string.byte(s, i)`: Extracts the byte value at position i in string s.
- `string.char(...)`: Creates a string from a given sequence of byte values.

```lua
local myByte = 72 -- ASCII code for 'H'
local myString = "Hello, world!"

local firstByte = string.byte(myString, 1) -- Extracts the first byte
print(firstByte) -- Output: 72

local newString = string.char(72, 101, 108, 108, 111) -- Creates "Hello"
print(newString) -- Output: Hello
```

<!-- TOC --><a href='#' name="handling-multi-byte-encoded-characters"></a>
## Handling Multi-byte Encoded Characters

Starting lua 5.3, `utf-8` support is [natively added](https://www.lua.org/manual/5.3/manual.html#6.5){:target="_blank"}. If we need utf-8 support for earlier versions, we need to use external libraries like [luautf8](https://luarocks.org/modules/xavier-wang/luautf8){:target="_blank}

Using `utf8.codes`, we can print individual unicode characters and get length of the total characters in utf-8 string.
```lua
> x = "ÆØÅ"
> for _, c in utf8.codes(x) do
>>   print(utf8.char(c))
>> end
Æ
Ø
Å
> utf8.len(x)
3
> string.len(x)
6
> x=😁
> for _, c in utf8.codes(x) do 
>> print(utf8.char(c))
>> end
😁
> utf8.len(x)
1
> string.len(x)
4
```

Also, note that we can print individual bytes that make up this 4 byte chacracter, but as explained before, this character is not really a combination of these 4 single bytes.

```lua
> for i=1, #x do
>>   print(x:byte(i))
>> end
240
159
152
129
> for i=1, #x do print(utf8.char(x:byte(i))) end
ɀ
ř
Œ
Ĩ
```

<!-- TOC --><a href='#' name="conclusion"></a>
## Conclusion

Lua's treatment of strings as byte sequences simplifies many programming tasks, making operations more straightforward and efficient. However, this approach requires careful handling of multi-byte characters and encoding schemes, particularly in global applications. By leveraging Lua's built-in functions and possibly supplementing with external libraries for complex character encodings, developers can overcome these challenges. I hope this article gave enough of a base for anyone looking to understand this topic.

<!-- TOC --><a href='#' name="references"></a>
## References

- [Lua Manual for strings](https://www.lua.org/pil/20.html){:target="_blank"}

- [Lua Manual for utf-8 support](https://www.lua.org/manual/5.3/manual.html#6.5){:target="_blank"}

If you want to read more on unicode, utf-8 and encodings, I would highly recommend the following 3:

- [The OG article](https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/){:target="_blank"}

- [Very informative article on UTF-8](https://blog.hubspot.com/website/what-is-utf-8#:~:text=UTF%2D8%20is%20an%20encoding,or%20%E2%80%9CUnicode%20Transformation%20Format.%E2%80%9D){:target="_blank"}

- [Very informative article on Unicode Working](https://deliciousbrains.com/how-unicode-works/){:target="_blank"}


<!-- TOC --><a href='#' name="appendix"></a>
## Appendix

<!-- TOC --><a href='#' name="brief-on-utf-8-encoding-basics"></a>
### Brief on UTF-8 Encoding Basics:
`UTF-8` encodes Unicode characters using 1 to 4 `bytes`, depending on the character’s Unicode code point. The first 128 characters (`US-ASCII`) need just one `byte`. Characters with higher code points require more bytes.

In UTF-8, the number of bytes used for encoding a character determines the pattern of bits in those bytes.
- `1-byte` characters are straightforward: `0xxxxxxx`. This covers standard ASCII.
- `2-byte` characters follow this pattern: `110xxxxx 10xxxxxx`.
- `3-byte` characters follow: `1110xxxx 10xxxxxx 10xxxxxx`.
- `4-byte` characters follow: `11110xxx 10xxxxxx 10xxxxxx 10xxxxxx`.

**Individual Bytes vs. Complete Characters**
When `UTF-8` encodes characters that require multiple bytes, each byte in the sequence is specialized:

The first byte indicates how many bytes in total will represent the character and starts the character encoding.
Subsequent bytes in the sequence (beginning with 10) are continuation bytes. These bytes do not make sense on their own as independent characters.

Consider the character `é`, which is represented in `UTF-8` as two bytes: `C3 A9`.

`C3` on its own does not represent any character because it is expecting a continuation byte.
`A9` is also not a standalone character in `UTF-8`; it must follow a byte like `C3` to complete the character.