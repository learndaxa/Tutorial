---
title: Creating an instance
description: Creating an instance
slug: "getting-started/creating-an-instance"
editUrl: https://github.com/learndaxa/Tutorial/edit/main/docs/03 Getting started/01_Creating_an_instance.md
---

## Include the header

Well done! We have now created all the fundamentals and can finally start using Daxa. To use Daxa's C++ API, we need to include the following header:

```cpp
#include <daxa/daxa.hpp>
```

Since this header is already included in the window.hpp header file, we don't need to also include this in our main.cpp

## Create a new instance

Using this header we can now generate a new Daxa instance!

```cpp
daxa::Instance instance = daxa::create_instance({});
```

A curious thing you will notice is that most function calls take a struct as a parameter. This is done to emulate named parameters and also to enable out-of-order default argument values using C++20"s designated initializers.

Daxa is a relatively explicit API but has a lot of defaults via struct default member values. This makes it much nicer to use in many cases.

Nearly all Daxa objects can be assigned a debug name in creation. This name is used in the error messages we emit and is also displayed in tools like RenderDoc.
