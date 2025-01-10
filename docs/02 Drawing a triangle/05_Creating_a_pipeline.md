---
title: Creating a pipeline
description: Creating a pipeline
slug: "drawing-a-triangle/creating-a-pipeline"
editUrl: https://github.com/learndaxa/Tutorial/edit/main/docs/02 Drawing a triangle/05_Creating_a_pipeline.md
---

## Pipeline manager

In this tutorial, we will be using the pipeline manager, which is an additional Daxa feature that has to be explicitly imported with the header `<daxa/utils/pipeline_manager.hpp>` and also has to be enabled in our Vcpkg manifest. Both of these steps are already done in the sample code.

A pipeline manager is responsible for managing and executing different render pipelines which we will define later on. This replaces the traditional command recording.

We have to provide the pipeline manager with the device we want to use, our shader directories as well as the shader language. In this tutorial, we will be using GLSL.

```cpp
auto pipeline_manager = daxa::PipelineManager({
    .device = device,
    .shader_compile_options = {
        .root_paths = {
            DAXA_SHADER_INCLUDE_DIR,
            "./src/shader",
        },
        .language = daxa::ShaderLanguage::GLSL,
        .enable_debug_info = true,
    },
    .name = "my pipeline manager",
});
```

## Rasterization Pipeline

We now can create our first pipeline. For a rasterization pipeline, we need to provide the shaders we want to use, the color attachments (Similar to the [OpenGL g-buffers](https://learnopengl.com/Advanced-Lighting/Deferred-Shading)) and the size of our push constant (We will explain this in the next section).

```cpp
std::shared_ptr<daxa::RasterPipeline> pipeline;
{
    auto result = pipeline_manager.add_raster_pipeline({
        .vertex_shader_info = daxa::ShaderCompileInfo{.source = daxa::ShaderFile{"main.glsl"}},
        .fragment_shader_info = daxa::ShaderCompileInfo{.source = daxa::ShaderFile{"main.glsl"}},
        .color_attachments = {{.format = swapchain.get_format()}},
        .raster = {},
        .push_constant_size = sizeof(MyPushConstant),
        .name = "my pipeline",
    });
    if (result.is_err())
    {
        std::cerr << result.message() << std::endl;
        return -1;
    }
    pipeline = result.value();
}
```
