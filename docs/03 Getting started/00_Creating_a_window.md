---
title: Creating a Window
description: Learn how to set up a blank window for your graphics application using GLFW and Daxa.
slug: "drawing-a-triangle/creating-a-window"
editUrl: https://github.com/learndaxa/Tutorial/edit/main/docs/02 Drawing a triangle/00_Creating_a_window.md
---

## Introduction

Creating a window is the first step when developing a graphics application. This tutorial walks you through creating a `window.hpp` file to manage a window using GLFW and integrate it with Daxa.

---

## 1. Creating the Header File

Start by creating a new header file `src/window.hpp`. This file will serve as the abstraction layer for the GLFW windowing library, encapsulating its functionalities within a clean and reusable interface.

```cpp
// window.hpp
#pragma once

struct AppWindow {
    // Window-related properties and methods will go here
};
```

<details>
<summary>Explanation</summary>

- `#pragma once`: Ensures the file is included only once during compilation, preventing duplicate definitions.
- `struct AppWindow`: Declares a placeholder structure to encapsulate window-related functionality. This allows us to modularly add methods and properties later.

</details>

## 2. Including Required Libraries

To integrate GLFW and Daxa, include the necessary headers. Since GLFW provides platform-specific APIs for window creation and management, use preprocessor directives to include platform-specific definitions.

```diff lang="cpp"
// window.hpp
#pragma once

+#include <daxa/daxa.hpp> 
+using namespace daxa::types; // For types like `u32`
+
+#include <GLFW/glfw3.h>
+
+#if defined(_WIN32)
+    #define GLFW_EXPOSE_NATIVE_WIN32
+    #define GLFW_NATIVE_INCLUDE_NONE
+    using HWND = void*;
+#elif defined(__linux__)
+    #define GLFW_EXPOSE_NATIVE_X11
+    #define GLFW_EXPOSE_NATIVE_WAYLAND
+#endif
+
+#include <GLFW/glfw3native.h> // Platform-specific GLFW functions

struct AppWindow {
    // Window-related properties and methods will go here
};
```

<details>
<summary>Why These Includes?</summary>

- **Daxa**: Provides rendering capabilities and essential types (like `u32` for unsigned 32-bit integers).
- **GLFW**: A cross-platform library for creating windows and handling input.
- **GLFW Native Headers**: Expose low-level platform-specific APIs (e.g., Windows HWND or Linux X11). :::

</details>

## 3. Defining Window Properties

Define the properties necessary to manage the window's state, such as dimensions, a pointer to the GLFW window object, and flags to track the window's status.

```diff lang="cpp"
// window.hpp
struct AppWindow {
+    GLFWwindow* glfw_window_ptr; // Pointer to the GLFW window object
+    u32 width, height;           // Dimensions of the window
+    bool minimized = false;      // Tracks if the window is minimized
+    bool swapchain_out_of_date = false; // Tracks if the swapchain needs updating
};
```

<details>
<summary>Explanation</summary>

1. `GLFWwindow* glfw_window_ptr`: Stores the GLFW window object created during initialization. This is the primary handle for interacting with the window.
2. `u32 width, height`: Tracks the window's current width and height, useful for managing rendering surfaces and responding to resize events.
3. `bool minimized`: A flag to check if the window is minimized. Rendering can often be paused when minimized to save resources.
4. `bool swapchain_out_of_date`: Indicates when the swapchain (used for rendering) needs to be recreated, often after window resizing.

</details>

:::tip[Managing Window State]
Always validate the window's state before performing rendering operations to avoid undefined behavior.
Use the swapchain_out_of_date flag to trigger re-creation of Vulkan/Daxa rendering resources during resize events.
:::

## 4. Constructor and Destructor

In this step, you'll define the **constructor** to initialize the window, configure callbacks for resizing, and handle cleanup in the **destructor**.

### Constructor

The constructor initializes the GLFW window, sets its properties, and registers callbacks for resizing events.

```cpp
// window.hpp
explicit AppWindow(char const* window_name, u32 sx = 800, u32 sy = 600) 
    : width{sx}, height{sy} {
    glfwInit(); // Initialize GLFW

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API); // No graphics API
    glfwWindowHint(GLFW_RESIZABLE, GLFW_TRUE);    // Make the window resizable

    // Create the GLFW window
    glfw_window_ptr = glfwCreateWindow(
        static_cast<i32>(width), static_cast<i32>(height), 
        window_name, nullptr, nullptr
    );

    // Associate the AppWindow object with the GLFW window
    glfwSetWindowUserPointer(glfw_window_ptr, this);

    // Set a callback to handle resizing events
    glfwSetWindowSizeCallback(glfw_window_ptr, [](GLFWwindow* window, int size_x, int size_y) {
        auto* win = static_cast<AppWindow*>(glfwGetWindowUserPointer(window));
        win->width = static_cast<u32>(size_x);
        win->height = static_cast<u32>(size_y);
        win->swapchain_out_of_date = true;
    });
}
```

<details>
<summary>Key Points</summary>

1. `glfwInit()`: Initializes the GLFW library. Always call this before using GLFW functions.
2. `glfwWindowHint()`: Configures the window. Setting `GLFW_CLIENT_API` to `GLFW_NO_API` means the window won't automatically use a graphics API like OpenGL.
3. `glfwSetWindowUserPointer()`: Associates a user-defined pointer (our `AppWindow` instance) with the GLFW window, allowing us to reference the `AppWindow` object in callbacks.
4. **Resize Callback**: Updates window dimensions and marks the swapchain as out-of-date whenever the window size changes.

</details>

:::tip[Resize Callback Use Case]
This callback is crucial for Vulkan-based rendering. Resizing a window invalidates the swapchain, and marking it as out-of-date helps ensure rendering resources are re-created correctly.
:::

### Destructor

The destructor ensures proper cleanup of GLFW resources to avoid memory leaks or dangling pointers.

```cpp
// window.hpp
~AppWindow() {
    glfwDestroyWindow(glfw_window_ptr); // Destroy the GLFW window
    glfwTerminate();                    // Terminate GLFW
}
```

<details>
<summary>Why This is Important</summary>

- `glfwDestroyWindow()`: Releases memory and handles associated with the window.
- `glfwTerminate()`: Cleans up GLFW internals. Failing to call this can lead to resource leaks.

</details>

## 5. Native Handles and Platform Identification

To interface with platform-specific window systems (e.g., Windows' HWND or Linux's X11/Wayland), expose methods to retrieve native handles and identify the platform.

### Native Handle Retrieval

This function returns a native window handle compatible with Daxa's `NativeWindowHandle` type.

```cpp
// window.hpp
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
```

<details>
<summary>Key Notes</summary>

1. **Platform-Specific Retrieval**:
    - On Windows, `glfwGetWin32Window()` retrieves an `HWND` handle.
    - On Linux, either `glfwGetX11Window()` or `glfwGetWaylandWindow()` is used, depending on the platform.

2. `daxa::NativeWindowHandle`: A Daxa-specific abstraction for handling different windowing systems. By returning this type, the function ensures compatibility with Daxa's rendering APIs.

</details>

### Platform Identification

This static function identifies the native platform and maps it to Daxa's NativeWindowPlatform enumeration.

```cpp
// window.hpp
static auto get_native_platform() -> daxa::NativeWindowPlatform {
    switch (glfwGetPlatform()) {
        case GLFW_PLATFORM_WIN32: return daxa::NativeWindowPlatform::WIN32_API;
        case GLFW_PLATFORM_X11: return daxa::NativeWindowPlatform::XLIB_API;
        case GLFW_PLATFORM_WAYLAND: return daxa::NativeWindowPlatform::WAYLAND_API;
        default: return daxa::NativeWindowPlatform::UNKNOWN;
    }
}
```

<details>
<summary>Key Notes</summary>

1. `glfwGetPlatform()`: Returns the current platform (e.g., Win32, X11, Wayland).
2. ***Mapping to `daxa::NativeWindowPlatform`**: Provides an abstraction layer to decouple GLFW-specific platform identifiers from Daxa's API.

</details>

## 6. Utility Functions

To simplify window management, implement utility methods for common tasks like mouse control, checking if the window should close, and handling updates.

```cpp
// window.hpp
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

inline GLFWwindow* get_glfw_window() const {
    return glfw_window_ptr;
}
```

<details>
<summary>Function Breakdown</summary>

1. `set_mouse_capture(bool should_capture)`:
    - Centers the mouse cursor within the window and adjusts its behavior.
    - `GLFW_CURSOR_DISABLED`: Locks the cursor to the window, often used in first-person or immersive applications.
    - `GLFW_CURSOR_NORMAL`: Frees the cursor for standard interaction.
    - `GLFW_RAW_MOUSE_MOTION`: Enables raw mouse motion, bypassing OS-level acceleration for precise control. This function is ideal for games or interactive simulations where precise control over the mouse is required.
2. `should_close()`:
    - Returns whether the window should close, typically triggered by the user clicking the close button or pressing Alt+F4.
      :::info[Why Check for Closure?]
      This is a standard condition for exiting the main application loop and ensures proper cleanup before termination.
      :::
3. `update()`:
    - Handles event polling and buffer swapping.
    - `glfwPollEvents()`: Processes all pending input events (e.g., mouse movement, keyboard presses).
    - `glfwSwapBuffers()`: Swaps the front and back buffers to present the rendered frame. Always call this function once per frame to ensure smooth rendering and responsive input handling.
4. `get_glfw_window()`:
    - Provides direct access to the raw GLFWwindow* pointer for advanced interactions or integrations.

</details>

## 7. Opening the Window

Integrate the `AppWindow` class into your application by creating and managing the window in `main.cpp`.

### Example Usage in main.cpp

```cpp
// main.cpp
#include "window.hpp"

int main(int argc, char const* argv[]) {
    // Create a window
    auto window = AppWindow("Learn Daxa", 860, 640);

    // Daxa rendering initialization code goes here...

    // Main loop
    while (!window.should_close()) {
        window.update();
    }

    return 0;
}
```

<details>
<summary>Step-by-Step Explanation</summary>

1. Creating the Window:
    - `AppWindow("Learn Daxa", 860, 640)` initializes a window with the title `"Learn Daxa"` and dimensions 860x640.

2. Main Application Loop:
    - The `while (!window.should_close())` loop keeps the application running until the window is closed.
    - `window.update()` ensures the application processes events and renders frames.

3. Integration with Daxa:
    - Add Daxa-specific rendering code (e.g., setting up pipelines, rendering frames) in place of the `// Daxa rendering initialization code goes here...` comment.

</details>

## Summary

You now have a fully functional GLFW window integrated with Daxa, complete with resizing support and utility functions. Use this as the foundation for your graphics applications.
