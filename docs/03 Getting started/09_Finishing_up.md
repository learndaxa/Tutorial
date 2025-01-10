---
title: Finishing up
description: Finishing up
slug: "drawing-a-triangle/finishing-up"
editUrl: https://github.com/learndaxa/Tutorial/edit/main/docs/02 Drawing a triangle/09_Finishing_up.md
---

## Implementing the main loop

```cpp
while (!window.should_close()){
    window.update();

    if (window.swapchain_out_of_date){
        swapchain.resize();
        window.swapchain_out_of_date = false;
    }

    // acquire the next image
    auto swapchain_image = swapchain.acquire_next_image();
    if (swapchain_image.is_empty())
    {
        continue;
    }

    // We update the image id of the task swapchain image.
    task_swapchain_image.set_images({.images = std::span{&swapchain_image, 1}});

    // So, now all we need to do is execute our task graph!
    loop_task_graph.execute({});
    device.collect_garbage();
}
```

## Cleaning up

Finally, we can clean up!

```cpp
device.destroy_buffer(buffer_id);

device.wait_idle();
device.collect_garbage();
```

## Running the code

You have now completed the Daxa tutorial! If you now run the code, you should have a triangle appearing in the window!
Running the code with the VSCode debugger should be as simple as pressing the debug button, though you may need to create a launch.json if the working directory is wrong.

Otherwise, you can manually run the CMake commands to configure, build, and then run the executable directly like so:
```
cmake --preset=Debug
cmake --build build/Debug
./build/Debug/learndaxa.exe
```

THE APPLICATION MUST USE THE REPO ROOT DIRECTORY FOR THE SHADER RELATIVE PATHS TO WORK PROPERLY

## Final Code

```cpp
// src/main.cpp

#include "window.hpp"
#include "shader/shared.inl"

#include <daxa/utils/pipeline_manager.hpp>
#include <daxa/utils/task_graph.hpp>

void upload_vertex_data_task(daxa::TaskGraph & tg, daxa::TaskBufferView vertices)
{
    tg.add_task({
        .attachments = {
            daxa::inl_attachment(daxa::TaskBufferAccess::TRANSFER_WRITE, vertices),
        },
        .task = [=](daxa::TaskInterface ti)
        {
            auto data = std::array{
                MyVertex{.position = {-0.5f, +0.5f, 0.0f}, .color = {1.0f, 0.0f, 0.0f}},
                MyVertex{.position = {+0.5f, +0.5f, 0.0f}, .color = {0.0f, 1.0f, 0.0f}},
                MyVertex{.position = {+0.0f, -0.5f, 0.0f}, .color = {0.0f, 0.0f, 1.0f}},
            };
            auto staging_buffer_id = ti.device.create_buffer({
                .size = sizeof(data),
                .allocate_info = daxa::MemoryFlagBits::HOST_ACCESS_RANDOM,
                .name = "my staging buffer",
            });
            ti.recorder.destroy_buffer_deferred(staging_buffer_id);
            auto * buffer_ptr = ti.device.buffer_host_address_as<std::array<MyVertex, 3>>(staging_buffer_id).value();
            *buffer_ptr = data;
            ti.recorder.copy_buffer_to_buffer({
                .src_buffer = staging_buffer_id,
                .dst_buffer = ti.get(vertices).ids[0],
                .size = sizeof(data),
            });
        },
        .name = "upload vertices",
    });
}

void draw_vertices_task(daxa::TaskGraph & tg, std::shared_ptr<daxa::RasterPipeline> pipeline, daxa::TaskBufferView vertices, daxa::TaskImageView render_target)
{
    tg.add_task({
        .attachments = {
            daxa::inl_attachment(daxa::TaskBufferAccess::VERTEX_SHADER_READ, vertices),
            daxa::inl_attachment(daxa::TaskImageAccess::COLOR_ATTACHMENT, daxa::ImageViewType::REGULAR_2D, render_target),
        },
        .task = [=](daxa::TaskInterface ti)
        {
            auto const size = ti.device.info(ti.get(render_target).ids[0]).value().size;

            daxa::RenderCommandRecorder render_recorder = std::move(ti.recorder).begin_renderpass({
                .color_attachments = std::array{
                    daxa::RenderAttachmentInfo{
                        .image_view = ti.get(render_target).view_ids[0],
                        .load_op = daxa::AttachmentLoadOp::CLEAR,
                        .clear_value = std::array<daxa::f32, 4>{0.1f, 0.0f, 0.5f, 1.0f},
                    },
                },
                .render_area = {.width = size.x, .height = size.y},
            });

            render_recorder.set_pipeline(*pipeline);
            render_recorder.push_constant(MyPushConstant{
                .my_vertex_ptr = ti.device.device_address(ti.get(vertices).ids[0]).value(),
            });
            render_recorder.draw({.vertex_count = 3});
            ti.recorder = std::move(render_recorder).end_renderpass();
        },
        .name = "draw vertices",
    });
}

int main(int argc, char const *argv[])
{
    // Create a window
    auto window = AppWindow("Learn Daxa", 860, 640);

    daxa::Instance instance = daxa::create_instance({});

    daxa::Device device = instance.create_device_2(instance.choose_device({}, {}));

    daxa::Swapchain swapchain = device.create_swapchain({
        .native_window = window.get_native_handle(),
        .native_window_platform = window.get_native_platform(),
        .present_mode = daxa::PresentMode::FIFO,
        .image_usage = daxa::ImageUsageFlagBits::TRANSFER_DST,
        .name = "my swapchain",
    });

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

    auto buffer_id = device.create_buffer({
        .size = sizeof(MyVertex) * 3,
        .name = "my vertex data",
    });

    auto task_swapchain_image = daxa::TaskImage{{.swapchain_image = true, .name = "swapchain image"}};
    auto task_vertex_buffer = daxa::TaskBuffer({
        .initial_buffers = {.buffers = std::span{&buffer_id, 1}},
        .name = "task vertex buffer",
    });

    auto loop_task_graph = daxa::TaskGraph({
        .device = device,
        .swapchain = swapchain,
        .name = "loop",
    });
    loop_task_graph.use_persistent_buffer(task_vertex_buffer);
    loop_task_graph.use_persistent_image(task_swapchain_image);
    draw_vertices_task(loop_task_graph, pipeline, task_vertex_buffer, task_swapchain_image);

    loop_task_graph.submit({});
    // And tell the task graph to do the present step.
    loop_task_graph.present({});
    // Finally, we complete the task graph, which essentially compiles the
    // dependency graph between tasks, and inserts the most optimal synchronization!
    loop_task_graph.complete({});

    {
        auto upload_task_graph = daxa::TaskGraph({
            .device = device,
            .name = "upload",
        });

        upload_task_graph.use_persistent_buffer(task_vertex_buffer);

        upload_vertex_data_task(upload_task_graph, task_vertex_buffer);

        upload_task_graph.submit({});
        upload_task_graph.complete({});
        upload_task_graph.execute({});
    }

    while (!window.should_close()){
        window.update();

        if (window.swapchain_out_of_date){
            swapchain.resize();
            window.swapchain_out_of_date = false;
        }

        // acquire the next image
        auto swapchain_image = swapchain.acquire_next_image();
        if (swapchain_image.is_empty())
        {
            continue;
        }

        // We update the image id of the task swapchain image.
        task_swapchain_image.set_images({.images = std::span{&swapchain_image, 1}});

        // So, now all we need to do is execute our task graph!
        loop_task_graph.execute({});
        device.collect_garbage();
    }

    device.destroy_buffer(buffer_id);

    device.wait_idle();
    device.collect_garbage();

    return 0;
}
```

```cpp
// src/window.hpp

#pragma once

#include <daxa/daxa.hpp>
using namespace daxa::types;

#include <GLFW/glfw3.h>
#if defined(_WIN32)
#define GLFW_EXPOSE_NATIVE_WIN32
#define GLFW_NATIVE_INCLUDE_NONE
using HWND = void *;
#elif defined(__linux__) /
#define GLFW_EXPOSE_NATIVE_X11
#define GLFW_EXPOSE_NATIVE_WAYLAND
#endif
#include <GLFW/glfw3native.h>

struct AppWindow {
    GLFWwindow *glfw_window_ptr;
    u32 width, height;
    bool minimized = false;
    bool swapchain_out_of_date = false;

    explicit AppWindow(char const *window_name, u32 sx = 800, u32 sy = 600) : width{sx}, height{sy} {
        // Initialize GLFW
        glfwInit();

        // Tell GLFW to not include any other API
        glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);

        // Tell GLFW to make the window resizable
        glfwWindowHint(GLFW_RESIZABLE, GLFW_TRUE);

        // Create the window
        glfw_window_ptr = glfwCreateWindow(static_cast<i32>(width), static_cast<i32>(height), window_name, nullptr, nullptr);

        // Set the user pointer to this window
        glfwSetWindowUserPointer(glfw_window_ptr, this);

        // When the window is resized, update the width and height and mark the swapchain as out of date
        glfwSetWindowSizeCallback(glfw_window_ptr, [](GLFWwindow *window, int size_x, int size_y) {
            auto *win = static_cast<AppWindow *>(glfwGetWindowUserPointer(window));
            win->width = static_cast<u32>(size_x);
            win->height = static_cast<u32>(size_y);
            win->swapchain_out_of_date = true;
        });
    }

    ~AppWindow() {
        glfwDestroyWindow(glfw_window_ptr);
        glfwTerminate();
    }

    auto get_native_handle() const -> daxa::NativeWindowHandle {
#if defined(_WIN32)
        return glfwGetWin32Window(glfw_window_ptr);
#elif defined(__linux__)
        switch (get_native_platform()) {
        case daxa::NativeWindowPlatform::WAYLAND_API:
            return reinterpret_cast<daxa::NativeWindowHandle>(glfwGetWaylandWindow(glfw_window_ptr));
        case daxa::NativeWindowPlatform::XLIB_API:
        default:
            return reinterpret_cast<daxa::NativeWindowHandle>(glfwGetX11Window(glfw_window_ptr));
        }
#endif
    }

    static auto get_native_platform() -> daxa::NativeWindowPlatform {
        switch (glfwGetPlatform()) {
        case GLFW_PLATFORM_WIN32: return daxa::NativeWindowPlatform::WIN32_API;
        case GLFW_PLATFORM_X11: return daxa::NativeWindowPlatform::XLIB_API;
        case GLFW_PLATFORM_WAYLAND: return daxa::NativeWindowPlatform::WAYLAND_API;
        default: return daxa::NativeWindowPlatform::UNKNOWN;
        }
    }

    inline void set_mouse_capture(bool should_capture) const {
        glfwSetCursorPos(glfw_window_ptr, static_cast<f64>(width / 2.), static_cast<f64>(height / 2.));
        glfwSetInputMode(glfw_window_ptr, GLFW_CURSOR, should_capture ? GLFW_CURSOR_DISABLED : GLFW_CURSOR_NORMAL);
        glfwSetInputMode(glfw_window_ptr, GLFW_RAW_MOUSE_MOTION, should_capture);
    }

    inline bool should_close() const {
        return glfwWindowShouldClose(glfw_window_ptr);
    }

    inline void update() const {
        glfwPollEvents();
        glfwSwapBuffers(glfw_window_ptr);
    }

    inline GLFWwindow *get_glfw_window() const {
        return glfw_window_ptr;
    }

    inline bool should_close() {
        return glfwWindowShouldClose(glfw_window_ptr);
    }
};
```

```cpp
// src/shaders/shared.inl

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

```cpp
// src/shaders/main.glsl

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
