---
layout: post
title:  "The silliness of Excel XOR function"
tags: excel
---

<!-- References -->
[boolean-ops]: <https://en.wikipedia.org/wiki/Boolean_algebra#Operations>
[exclusive-or]: <https://en.wikipedia.org/wiki/Exclusive_or>
[and]: <https://support.microsoft.com/en-us/office/and-function-5f19b2e8-e1df-4408-897a-ce285a19e9d9>
[or]: <https://support.microsoft.com/en-us/office/or-function-7d17ad14-8700-4281-b308-00b131e22af0>
[xor]: <https://support.microsoft.com/en-us/office/xor-function-1548d4c2-5e47-4f77-9a92-0533bba14f37>
<!-- End References -->

Like all programming languages, Excel provides the usual arsenal of [Boolean operations][boolean-ops]: [AND][and], [OR][or] and [XOR][xor]. Unlike most programming languages, however, Excel allows these to have more than 2 arguments. 

For AND and OR this is a no-brainer. The natural meaning of AND: "all of the arguments must be true" trivially extends to any number of arguments. Similarly for OR which means "at least one of the arguments must be true". 

But what is the meaning of XOR of more than 2 arguments? For two arguments the standard definition of [Exclusive OR][exclusive-or] is:

> one or the other but not both

With more than 2 arguments this could usefully be extended as either:

1. one and only one of arguments must be true ("only one of" for short)
2. one or more but not all of arguments must be true ("at least one but not all" for short)

The first alternative is arguably more useful since the condition it checks for is most often relevant for human affairs. You can choose only 1 time slot for a meeting out of N alternatives, only one coupon at checkout out of N available ones etc. etc.

The second alternative happens less often but still does occur and can be defended on consistency principles. 2 arguments `XOR(a,b)` is equivalent to `AND(OR(a, b), NOT(AND(a, b))` or if you prefer mathematical notation

a ⊻ b = (a ⋁ b) ⋀ ¬(a ⋀ b)

Extending the right hand side to more than 2 arguments:

(a ⋁ b ⋁ c ⋁ ...) ⋀ ¬(a ⋀ b ⋀ c ⋀ ...)

produces precisely the second definition.

However, whichever way you might prefer, what Excel is doing is **neither**!

Instead, Excel simply applies XOR operation sequentially to its arguments. That is for Excel `XOR(a, b, c)` means `XOR(XOR(a, b), c)` or in mathematical terms:

a ⊻ b ⊻ c ⊻ ...

Unfortunately, unlike what happens with AND and OR, doing it this way provides rather useless results that have nothing to do with either definition above. Observe that applying the operation sequentially results in:

`XOR(true, true, true) = true`

but 

`XOR(true, true, true, true) = false`

This is quite silly of course. 

Given the backward compatibility constraints I doubt the situation will ever be rectified. Until it is, however, using XOR formula in Excel with more than 2 arguments is probably not what you should be doing.

The usual disclaimer applies: all this is based on Excel on the Web behavior as of the date of this article. Nothing prevents Microsoft from changing behavior of XOR function at a later date.

