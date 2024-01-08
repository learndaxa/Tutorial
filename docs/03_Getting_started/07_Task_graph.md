## Description

While not entirely necessary, we're going to use TaskGraph, which allows us to compile a list of GPU tasks and their dependencies into a synchronized set of commands. This simplifies your code by making different tasks completely self-contained, while also generating the most optimal synchronization for the tasks you describe.

## Creating a vertex uploading task

Before we can use a task graph, we first need to create actual tasks that can be executed. The first task we are going to create will upload vertex data to the GPU.

Each task struct must consist of a child struct 'Uses' that will store all shared resources, as well as a callback function that gets called whenever the task is executed.

For our task, this base task structure will look like this:

```cpp
struct UploadVertexDataTask
{
    struct Uses
    {
        daxa::BufferTransferWrite vertex_buffer{};
    } uses = {};

    std::string_view name = "upload vertices";

    void callback(daxa::TaskInterface ti){
        // [...]
    }
};
```

In the callback function, we will first need to get the command recorder.

```cpp
auto & recorder = ti.get_recorder();
```

We also need the data we will upload. In this sample, we will use the standard triangle vertices.

```cpp
auto data = std::array{
    MyVertex{.position = {-0.5f, +0.5f, 0.0f}, .color = {1.0f, 0.0f, 0.0f}},
    MyVertex{.position = {+0.5f, +0.5f, 0.0f}, .color = {0.0f, 1.0f, 0.0f}},
    MyVertex{.position = {+0.0f, -0.5f, 0.0f}, .color = {0.0f, 0.0f, 1.0f}},
};
```

To send the data to the GPU, we can create a staging buffer, which has host access, so that we can then issue a command to copy from this buffer to the dedicated GPU memory.

```cpp
auto staging_buffer_id = ti.get_device().create_buffer({
    .size = sizeof(data),
    .allocate_info = daxa::MemoryFlagBits::HOST_ACCESS_RANDOM,
    .name = "my staging buffer",
});
```

We can also ask the command list to destroy this temporary buffer since we don't care about it living, but we DO need it to survive through its usage on the GPU (which won't happen until after these commands are submitted), so we tell the command list to destroy it in a deferred fashion.

```cpp
recorder.destroy_buffer_deferred(staging_buffer_id);
```

We then get the memory-mapped pointer of the staging buffer, and write the data directly to it.

```cpp
auto * buffer_ptr = ti.get_device().get_host_address_as<std::array<MyVertex, 3>>(staging_buffer_id).value();
*buffer_ptr = data;
recorder.copy_buffer_to_buffer({
    .src_buffer = staging_buffer_id,
    .dst_buffer = uses.vertex_buffer.buffer(),
    .size = sizeof(data),
});
```

## Creating a Rendering task

We will again create a base task:

```cpp
struct DrawToSwapchainTask
{
    struct Uses
    {
        // We declare a vertex buffer read. Later we assign the task vertex buffer handle to this use.
        daxa::BufferVertexShaderRead vertex_buffer{};
        // We declare a color target. We will assign the swapchain task image to this later.
        // The name `ImageColorAttachment<T_VIEW_TYPE = DEFAULT>` is a typedef for `daxa::TaskImageUse<daxa::TaskImageAccess::COLOR_ATTACHMENT, T_VIEW_TYPE>`.
        daxa::ImageColorAttachment<> color_target{};
    } uses = {};

    daxa::RasterPipeline * pipeline = {};

    std::string_view name = "draw task";

    void callback(daxa::TaskInterface ti){
        auto & recorder = ti.get_recorder();

        // [...]
    }
};
```

We first need to get the screen width and height in the callback function. We can do this by getting the target image dimensions.

```cpp
auto const size_x = ti.get_device().info_image(uses.color_target.image()).value().size.x;
auto const size_y = ti.get_device().info_image(uses.color_target.image()).value().size.y;
```

Next, we need to record an actual renderpass. The values are pretty self-explanatory if you have used OpenGL before. This contains the actual rendering logic.

```cpp
auto render_recorder = std::move(recorder).begin_renderpass({
    .color_attachments = std::array{
        daxa::RenderAttachmentInfo{
            .image_view = uses.color_target.view(),
            .load_op = daxa::AttachmentLoadOp::CLEAR,
            .clear_value = std::array<daxa::f32, 4>{0.1f, 0.0f, 0.5f, 1.0f},
        },
    },
    .render_area = {.x = 0, .y = 0, .width = size_x, .height = size_y},
});

render_recorder.set_pipeline(*pipeline);
render_recorder.push_constant(MyPushConstant{
    .my_vertex_ptr = ti.get_device().get_device_address(uses.vertex_buffer.buffer()).value(),
});
render_recorder.draw({.vertex_count = 3});
recorder = std::move(render_recorder).end_renderpass();
```

## Creating a Rendering TaskGraph

When using TaskGraph, we must create "virtual" resources (we call them task resources) whose usages are tracked, allowing for correct synchronization for them.

Back in our main method, the first we'll make is the swap chain image task resource. We could immediately give this task image an image ID. But in the case of the swapchain images we need to reacquire a new image every frame.

```cpp
auto task_swapchain_image = daxa::TaskImage{{.swapchain_image = true, .name = "swapchain image"}};
```

We will also create a buffer task resource, for our MyVertex buffer buffer_id. We do something a little special here, which is that we set the initial access of the buffer to be vertex shader read, and that's because we'll create a task list that will upload the buffer.

```cpp
auto task_vertex_buffer = daxa::TaskBuffer({
    .initial_buffers = {.buffers = std::span{&buffer_id, 1}},
    .name = "task vertex buffer",
});
```

Next, we need to create the actual task graph itself:

```cpp
auto loop_task_graph = daxa::TaskGraph({
    .device = device,
    .swapchain = swapchain,
    .name = "loop",
});
```

We need to explicitly declare all uses of persistent task resources because manually marking used resources makes it possible to detect errors in your graph recording.

```cpp
loop_task_graph.use_persistent_buffer(task_vertex_buffer);
loop_task_graph.use_persistent_image(task_swapchain_image);
```

Since we need the task graph to do something, we add a task that draws to the screen:

```cpp
loop_task_graph.add_task(DrawToSwapchainTask{
    .uses = {
        .vertex_buffer = task_vertex_buffer.view(),
        .color_target = task_swapchain_image.view(),
    },
    .pipeline = pipeline.get(),
});
```

Once we have added all the tasks we want, we have to tell the task graph we are done.

```cpp
loop_task_graph.submit({});
// And tell the task graph to do the present step.
loop_task_graph.present({});
// Finally, we complete the task graph, which essentially compiles the
// dependency graph between tasks, and inserts the most optimal synchronization!
loop_task_graph.complete({});
```

We have now created a new task graph that can simply repeat the steps it was given,

## Creating a vertex uploading TaskGraph

Now we record a secondary task graph, that is only executed once (in our sample code). This task graph uploads data to the 'vertex buffer'. Task Graph resources automatically link between graphics at runtime, so you don't need to be concerned about the synchronization of the vertex buffer between the two graphs.

Because this is only executed once, we can define it in a separate context.

```cpp
{
        auto upload_task_graph = daxa::TaskGraph({
            .device = device,
            .name = "upload",
        });

        upload_task_graph.use_persistent_buffer(task_vertex_buffer);

        upload_task_graph.add_task(UploadVertexDataTask{
            .uses = {
                .vertex_buffer = task_vertex_buffer.view(),
            },
        });

        upload_task_graph.submit({});
        upload_task_graph.complete({});
        upload_task_graph.execute({});
    }
```
