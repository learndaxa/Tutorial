## Description

The first thing when developing a graphics application is to open a blank window.

## Creating a header file

In this tutorial, we will create a new header file `src/window.hpp` that is responsible for interfacing and abstracting the windowing library of our choice GLFW.

```cpp
#pragma once

struct AppWindow{
};
```

Next, we need to include the libraries. Since we need the native window handle later, we also need the native GLFW headers. Since those libraries are platform dependant though, we need some extra preprocessor magic.

```cpp
#include <daxa/daxa.hpp>
// For things like `u32`. Not necessary of course.
using namespace daxa::types;

#include <GLFW/glfw3.h>
#if defined(_WIN32)
#define GLFW_EXPOSE_NATIVE_WIN32
#define GLFW_NATIVE_INCLUDE_NONE
using HWND = void *;
#elif defined(__linux__)
#define GLFW_EXPOSE_NATIVE_X11
#define GLFW_EXPOSE_NATIVE_WAYLAND
#endif
#include <GLFW/glfw3native.h>
```

We now need to add some properties to our window struct. This includes the GLFW window pointer, the current width and height, whether the window is minimized and whether the swapchain is out of date due to the resizing of the window.

```cpp
GLFWwindow * glfw_window_ptr;
u32 width, height;
bool minimized = false;
bool swapchain_out_of_date = false;
```

We can now create a constructor and a destructor for the window.

```cpp
explicit AppWindow(char const * window_name, u32 sx = 800, u32 sy = 600) : width{sx}, height{sy} {
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
        auto* win = static_cast<AppWindow*>(glfwGetWindowUserPointer(window));
        win->width = static_cast<u32>(size_x);
        win->height = static_cast<u32>(size_y);
        win->swapchain_out_of_date = true;
    });
}

~AppWindow() {
    glfwDestroyWindow(glfw_window_ptr);
    glfwTerminate();
}
```

Next, we need to create functions to obtain the native handle of the window and the platform identifier.

```cpp
auto get_native_handle() const -> daxa::NativeWindowHandle
{
#if defined(_WIN32)
    return glfwGetWin32Window(glfw_window_ptr);
#elif defined(__linux__)
    switch (get_native_platform())
    {
        case daxa::NativeWindowPlatform::WAYLAND_API:
            return reinterpret_cast<daxa::NativeWindowHandle>(glfwGetWaylandWindow(glfw_window_ptr));
        case daxa::NativeWindowPlatform::XLIB_API:
        default:
            return reinterpret_cast<daxa::NativeWindowHandle>(glfwGetX11Window(glfw_window_ptr));
    }
#endif
}

static auto get_native_platform() -> daxa::NativeWindowPlatform
{
    switch(glfwGetPlatform())
    {
        case GLFW_PLATFORM_WIN32: return daxa::NativeWindowPlatform::WIN32_API;
        case GLFW_PLATFORM_X11: return daxa::NativeWindowPlatform::XLIB_API;
        case GLFW_PLATFORM_WAYLAND: return daxa::NativeWindowPlatform::WAYLAND_API;
        default: return daxa::NativeWindowPlatform::UNKNOWN;
    }
}
```

While we are at it, we can also create some utility functions.

```cpp
inline void set_mouse_capture(bool should_capture) const
{
    glfwSetCursorPos(glfw_window_ptr, static_cast<f64>(width / 2.), static_cast<f64>(height / 2.));
    glfwSetInputMode(glfw_window_ptr, GLFW_CURSOR, should_capture ? GLFW_CURSOR_DISABLED : GLFW_CURSOR_NORMAL);
    glfwSetInputMode(glfw_window_ptr, GLFW_RAW_MOUSE_MOTION, should_capture);
}

inline bool should_close() const
{
    return glfwWindowShouldClose(glfw_window_ptr);
}

inline void update() const
{
    glfwPollEvents();
    glfwSwapBuffers(glfw_window_ptr);
}

inline GLFWwindow* get_glfw_window() const{
    return glfw_window_ptr;
}

inline bool should_close() {
    return glfwWindowShouldClose(glfw_window_ptr);
}
```

## Opening our window

We can now go ahead and open our window for the first time. We therefore need to extend our main.cpp file.

```cpp
#include "window.hpp"

int main(int argc, char const *argv[])
{
    // Create a window
    auto window = AppWindow("Learn Daxa", 860, 640);

    // Daxa code goes here...

    while (!window.should_close())
    {
        window.update();
    }

    return 0;
}
```
