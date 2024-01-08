## Description

The Vulkan spec defines Push Constants as:
> A small bank of values writable via the API and accessible in shaders. Push constants allow the application to set values used in shaders without creating buffers or modifying and binding descriptor sets for each update.

## Implementation

To use push constants in our demo project, we need to create a new file: `shader/shared.inl` which will be a shared file between our main program and our shader file. Since Glsl is more or less a superset of basic C, we can use some code snippets in both languages.

Since this document is treated as a header file in our C++ code, we can simply insert `#pragma once` at the top to make sure it's only included once. We also need to include the Daxa (Shader) API directly beneath it: `#include <daxa/daxa.inl>`.

We can now start to define common structs, etc. In this case, we need to create a new struct 'MyVertex' that can be pushed to the GPU. Our basic vertices will have a position and color attribute.

```cpp
struct MyVertex
{
    daxa_f32vec3 position;
    daxa_f32vec3 color;
};
```

Below this, we have to allow the shader to use pointers to our newly created struct.

```cpp
DAXA_DECL_BUFFER_PTR(MyVertex)
```

The last step is to create the push constant. The push constant struct needs the attribute 'daxa_BufferPtr' that points to another struct object.

```cpp
struct MyPushConstant
{
    daxa_BufferPtr(MyVertex) my_vertex_ptr;
};
```

To use this file in our main.cpp, we need to include it at the top: `#include "shader/shared.inl"`

## Final code

```cpp
#pragma once

#include <daxa/daxa.inl>

struct MyVertex
{
    daxa_f32vec3 position;
    daxa_f32vec3 color;
};

DAXA_DECL_BUFFER_PTR(MyVertex)

struct MyPushConstant
{
    daxa_BufferPtr(MyVertex) my_vertex_ptr;
};
```
