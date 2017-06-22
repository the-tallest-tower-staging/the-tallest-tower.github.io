---
layout: post
title: "The Scrapheap"
date: 2017-06-21 18:00:00 -0800
categories: development, programming, tech
---

Mini-update: I just finished implementing a "scrapheap" to supplement the D garbage collector. By scrapheap, I mean:

- A 16 MB pre-allocated chunk of memory, acquired from the OS on startup and never released
- Other code can allocate into that chunk using a dead simple bump-the-pointer stack
- The stack is reset at the end of each frame, and so the 16MB is "freed" and available for use again

It's very desirable to be able to allocate small amounts of short-lived memory without having to think too much about it. For example, when parsing string input, I might do the following:

```
bool DoesItMatch()
{
    string[] optionWords = optionText.strip().removechars(".,!'").toLower().split();
    return optionWords == someOtherThing;
}
```

Being able to do all that string manupulation in one line iss really nice. I don't have to allocate buffers or wrangle character arrays, it's all taken care of. In D though, the dark side of this is that it allocates several times, even though once this function returns, I don't use any of the allocations ever again.

Enter the scrapheap.

To enable it, I add one line to the beginning of this scope:

```
bool DoesItMatch()
{
    mixin(ScopeScrapheap!());
    string[] optionWords = optionText.strip().removechars(".,!'").toLower().split();
    return optionWords == someOtherThing;
}
```

This mixin tells the GC that from now until the end of this scope, every allocation should be redirected to the scrapheap. Now all the allocations in this function are practically free (just the cost of bumping a pointer), and deallocation is free (resetting the pointer at the end of the frame). As long as I'm not expecting any of the allocations to outlive the frame, I'm in the clear.

For use cases like this, it's the best of both worlds. Convenient-to-write code, without any allocation or deallocation overhead (or contributing to GC pauses). And these use cases make up a pretty significant portion of the allocations that occur in the game each frame. The conventional wisdom is that "allocating is always going to be expensive", but evidently this doesn't have to be true.