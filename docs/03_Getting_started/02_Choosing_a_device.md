## Device selection

A PC can have multiple graphics cards. Unlike OpenGL, you need to manually select which GPU you want to use to perform calculations. Since each GPU is different Daxa provides a simple system to select the GPU best suited for your application. You can choose the considered parameters yourself and change how they are weighed against each other.

Below is a sample code to select a GPU based on location & VRAM.

```cpp
daxa::Device device = instance.create_device({
    .selector = [](daxa::DeviceProperties const & device_props) -> daxa::i32
    {
        daxa::i32 score = 0;
        switch (device_props.device_type)
        {
        case daxa::DeviceType::DISCRETE_GPU: score += 10000; break;
        case daxa::DeviceType::VIRTUAL_GPU: score += 1000; break;
        case daxa::DeviceType::INTEGRATED_GPU: score += 100; break;
        default: break;
        }
        score += static_cast<daxa::i32>(device_props.limits.max_memory_allocation_count / 100000);
        return score;
    },
    .name = "my device",
});
```

The struct daxa::DeviceProperties has many fields indicating different GPU attributes, such as Vulkan version, device type, ray tracing capabilities, ... ([Code reference](https://github.com/Ipotrick/Daxa/blob/master/include/daxa/device.hpp#L188))
