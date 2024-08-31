## Device selection

A PC can have multiple graphics cards. Unlike OpenGL, you need to manually select which GPU you want to use to perform calculations. Since each GPU is different Daxa provides a simple system to select the GPU best suited for your application. You can choose the considered parameters yourself and change how they are weighed against each other.

Below is sample code that selects the first device provided by the Vulkan driver (so long as it supports all of Daxa's required features). This means that if the user sets a device override in an application such as NVIDIA control panel, said device will be selected.

```cpp
daxa::Device device = instance.create_device_2(instance.choose_device({}, {}));
```

Usually, this is the desired behavior. If you want more control over device selection, see how this is done in [The device creation sample in the Daxa repository](https://github.com/Ipotrick/Daxa/blob/master/tests/2_daxa_api/2_device/main.cpp)
