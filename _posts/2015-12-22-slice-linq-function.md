---
layout: post
title: "Slice an enumerable in LINQ"
description: ""
category: "back-end"
tags: [LINQ]
---
{% include JB/setup %}

One of the useful methods I found missing from built-in LINQ functions is the ability to slice an enumerable into chunks... So here I include it, should be useful for anyone in the future:

<!--more-->

Pass `fixedSize=true` to ensure the last set also gets the same number of items in it.

```csharp
public static IEnumerable<IEnumerable<T>> Slice<T>(this IEnumerable<T> items, int size, bool fixedSize = false, T defaultValue = default(T))
{
    var start = 0;
    items = items.ToArray();
    var total = items.Count();
    while (start < total)
    {
        var res = items.Skip(start).Take(size).ToList();
        while (fixedSize && res.Count < size)
        {
            res.Add(defaultValue);
        }
        yield return res;

        start += size;
    }
}
```
