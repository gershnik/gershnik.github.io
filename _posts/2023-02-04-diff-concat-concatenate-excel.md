---
layout: post
title:  "Difference between CONCAT and CONCATENATE in Excel"
tags: excel
---

<!-- References -->
[concat]: <https://support.microsoft.com/en-us/office/concat-function-9b1a9a3f-94ff-41af-9736-694cbd6b4ca2>
[concatenate]: <https://support.microsoft.com/en-us/office/concatenate-function-8f8ae884-2ca8-4f7a-b093-75d702bea31d>
<!-- End References -->

Excel provides two seemingly identical functions: [CONCAT][concat] and [CONCATENATE][concatenate]. The official documentation for both doesn't really say what are the differences (if any) between the two other than the cryptic difference in descriptions:

For CONCAT: "Combines the text from multiple ranges and/or strings..."

For CONCATENATE: "Joins several text items into one text item"

As well as the following warning

> Important: In Excel 2016, Excel Mobile, and Excel for the web, this function has been replaced with the CONCAT function. Although the CONCATENATE function is still available for backward compatibility, you should consider using CONCAT from now on. This is because CONCATENATE may not be available in future versions of Excel.

Leaving the dire warning aside (as if they are going to break millions of existing spreadsheets by removing it!) what is the difference?

Turns out the descriptions above provide the important clue. CONCAT is an _aggregate_ function similar to SUM or AVERAGE. It accepts both scalar arguments (strings, numbers) and ranges/arrays. When given a range/array it will concatenate their elements into the result. 

On the other hand CONCATENATE is a _scalar_ function like AND or OR. When given a range/array argument it takes one element from it and generates an _array_ on output. 

To see what it means consider the following sheet:

![Data](/images/concat-concatenate-data.png)

Now if you put the following formula in A5: `=CONCAT(A2:A4, A1:C1)` the result will be

![Concat](/images/concat.png)

But if you enter `=CONCATENATE(A2:A4, A1:C1)` you will get

![Concatenate](/images/concatenate.png)

Hopefully this makes things clear. 

The usual disclaimer applies: all this is based on Excel on the Wb behavior as of the date of this article. Nothing prevents Microsoft from changing behavior of either or both functions at a later date.



