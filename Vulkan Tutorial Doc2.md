# vulkan tutorial on harmony

参考资料主要是 [vulkan tutorial](https://vulkan-tutorial.com/Drawing_a_triangle/Setup/Instance) 和 [vulkan guide](https://vkguide.dev/docs/new_chapter_0/code_walkthrough/)

前期主要是对vulkan的配置，模板代码居多，vulkan guide使用vkbootstrap来简化vulkan的初始化过程，但是这个库未适配鸿蒙平台，所以这里主要参考vulkan tutorial的代码。      

## 创建vulkan实例

```cpp
    VkApplicationInfo appInfo{};
    appInfo.sType = VK_STRUCTURE_TYPE_APPLICATION_INFO;
    appInfo.pNext = nullptr;
    appInfo.pApplicationName = "Hello Triangle";
    appInfo.applicationVersion = VK_MAKE_VERSION(1, 0, 0);
    appInfo.pEngineName = "No Engine";
    appInfo.engineVersion = VK_MAKE_VERSION(1, 0, 0);
    appInfo.apiVersion = VK_API_VERSION_1_0;
```

这是vulkan app的配置信息 [VkApplicationInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkApplicationInfo.html)
