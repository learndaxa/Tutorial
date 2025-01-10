---
title: Shader
description: Shader
slug: "drawing-a-triangle/shader"
editUrl: https://github.com/learndaxa/Tutorial/edit/main/docs/02 Drawing a triangle/08_Shader.md
---

## Description

Shaders are small programs running on the GPU. Most commonly are vertex- and fragment shaders. Both of these together take buffer data as input and output an image.

As this section does not contain any new information, we will only provide a small walkthrough of the sample shader. This should be copied to src/shader/main.glsl.

## Final code

```cpp
// Includes the daxa shader API
#include <daxa/daxa.inl>

// Enabled the extension GL_EXT_debug_printf
#extension GL_EXT_debug_printf : enable

// Includes our shared types we created earlier
#include <shared.inl>

// Enabled the push constant MyPushConstant we specified in shared.inl
DAXA_DECL_PUSH_CONSTANT(MyPushConstant, push)

// We can define the vertex & fragment shader in one single file
#if DAXA_SHADER_STAGE == DAXA_SHADER_STAGE_VERTEX

layout(location = 0) out daxa_f32vec3 v_col;
void main()
{
    MyVertex vert = deref(push.my_vertex_ptr[gl_VertexIndex]);
    gl_Position = daxa_f32vec4(vert.position, 1);
    v_col = vert.color;
}

#elif DAXA_SHADER_STAGE == DAXA_SHADER_STAGE_FRAGMENT

layout(location = 0) in daxa_f32vec3 v_col;
layout(location = 0) out daxa_f32vec4 color;
void main()
{
    color = daxa_f32vec4(v_col, 1);

    // Debug printf is not necessary, we just use it here to show how it can be used.
    // To be able to see the debug printf output, you need to open Vulkan Configurator and enable it there.
    debugPrintfEXT("test\n");
}

#endif
```
