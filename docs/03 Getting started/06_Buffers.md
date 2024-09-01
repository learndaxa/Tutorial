---
title: Buffers
description: Buffers
slug: "getting-started/buffers"
editUrl: https://github.com/learndaxa/Tutorial/edit/main/docs/03 Getting started/06_Buffers.md
---

## Description

When uploading data to the GPU, OpenGL used target-specific buffer targets (Vertex Object/Array Buffers, etc.). Daxa uses bindless buffers instead. This means a buffer isn't bound to one target only. One buffer can be used in all of these different bind targets and there is therefore only one buffer 'type'.

To create a buffer, we simply need the device the memory should be allocated on as well as the allocation size.

To allocate the data needed for our triangle vertex data we can simply create a new buffer:

```cpp
auto buffer_id = device.create_buffer({
    .size = sizeof(MyVertex) * 3,
    .name = "my vertex data",
});
```
