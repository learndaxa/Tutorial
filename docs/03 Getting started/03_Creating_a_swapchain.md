---
title: Creating a swapchain
description: Creating a swapchain
slug: "drawing-a-triangle/creating-a-swapchain"
editUrl: https://github.com/learndaxa/Tutorial/edit/main/docs/02 Drawing a triangle/03_Creating_a_swapchain.md
---

## Swapchain creation

In Daxa, the swapchain is a key element in rendering graphics, acting as a bridge between your application and the display. It's a collection of buffers used for displaying images on the screen. Unlike other/older APIs, Vulkan requires explicit management of these, which Daxa luckily handles for you.

The following code sample creates a new swapchain using a native window handle and a native window platform. Both of these values are supposed to be supplied by your windowing library of choice.

```cpp
daxa::Swapchain swapchain = device.create_swapchain({
    // this handle is given by the windowing API
    .native_window = native_window_handle,
    // The platform would also be retrieved from the windowing API,
    // or by hard-coding it depending on the OS.
    .native_window_platform = native_window_platform,
    // Here we can supply a user-defined surface format selection
    // function, to rate formats. If you don't care what format the
    // swapchain images are in, then you can just omit this argument
    // because it defaults to `daxa::default_format_score(...)`
    .surface_format_selector = [](daxa::Format format)
    {
        switch (format)
        {
        case daxa::Format::R8G8B8A8_UINT: return 100;
        default: return daxa::default_format_score(format);
        }
    },
    .present_mode = daxa::PresentMode::MAILBOX,
    .image_usage = daxa::ImageUsageFlagBits::TRANSFER_DST,
    .name = "my swapchain",
});
```

In the sample code, the native window handle can be obtained by 2 of the helper-functions we created `AppWindow::get_native_handle()` and `AppWindow::get_native_platform()`

### daxa::PresentMode

This defines how the rendered images are supplied to your screen. `daxa::PresentMode::FIFO` is the recommended default option for most use cases.

| mode | meaning |
| --- | --- |
| daxa::PresentMode::IMMEDIATE | Images submitted by your application are transferred to the screen right away, which may result in tearing |
| daxa::PresentMode::FIFO | The swapchain is a queue where the display takes an image from the front of the queue when the display is refreshed and the program inserts rendered images at the back of the queue. If the queue is full then the program has to wait. This is most similar to vertical sync as found in modern games. The moment that the display is refreshed is known as "vertical blank". |
| daxa::PresentMode::FIFO_RELAXED | This mode only differs from the previous one if the application is late and the queue was empty at the last vertical blank. Instead of waiting for the next vertical blank, the image is transferred right away when it finally arrives. This may result in visible tearing. |
| daxa::PresentMode::MAILBOX | This is another variation of the second mode. Instead of blocking the application when the queue is full, the images that are already queued are simply replaced with the newer ones. This mode can be used to render frames as fast as possible while still avoiding tearing, resulting in fewer latency issues than standard vertical sync. This is commonly known as "triple buffering", although the existence of three buffers alone does not necessarily mean that the framerate is unlocked. Although this may be desirable, it has limited support on AMD devices |

## Swapchain usage

You can now acquire a new swapchain image by later running

```cpp
daxa::ImageId swapchain_image = swapchain.acquire_next_image();
```

If all swapchain images are used in queued submissions to the GPU, the present call will block. The Swapchain will also control frames in flight. It controls the acquire, present and frames in flight semaphores. Each of those can be queried with a function from the swapchain.

## Final code

```cpp
#include "window.hpp"

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

    while (!window.should_close())
    {
        window.update();
    }

    return 0;
}
```
