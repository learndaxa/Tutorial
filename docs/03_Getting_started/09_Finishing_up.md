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
device.wait_idle();
device.collect_garbage();
device.destroy_buffer(buffer_id);
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
