## 一、创建实例

```C++
VkResult vkCreateInstance(
    const VkInstanceCreateInfo*                 pCreateInfo,
    const VkAllocationCallbacks*                pAllocator,
    VkInstance*                                 pInstance);
```

<br> <br>

### [VkInstanceCreateInfo](https://registry.khronos.org/vulkan/specs/latest/man/html/VkInstanceCreateInfo.html)--实例信息

```cpp
// Provided by VK_VERSION_1_0
typedef struct VkInstanceCreateInfo {
    VkStructureType             sType;
    const void*                 pNext;
    VkInstanceCreateFlags       flags;//非必填项
    const VkApplicationInfo*    pApplicationInfo;
    uint32_t                    enabledLayerCount;
    const char* const*          ppEnabledLayerNames;
    uint32_t                    enabledExtensionCount;
    const char* const*          ppEnabledExtensionNames;
} VkInstanceCreateInfo;
```

[app信息](https://registry.khronos.org/vulkan/specs/latest/man/html/VkApplicationInfo.html)——名称、版本、API version之类的

支持的[层（layers）](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#extendingvulkan-layers)——命令的调用链中对指定的环节插入函数（例如验证层，日志，跟踪，插件）

支持的[插件（extensions）](https://registry.khronos.org/vulkan/specs/latest/html/vkspec.html#extendingvulkan-instance-extensions)——非常复杂，可以提供新的命令，结构体和枚举，用于扩展vulkan的功能

<br> <br>

### VkAllocationCallbacks

内存分配函数——自选内存分配函数

<br> <br>

## 获取支持的layers和extensions

直接指定layers

```C++
const std::vector<const char*> validationLayers = {
    //用于作为window，展示内容
    "VK_LAYER_OHOS_surface"
};
bool checkValidationLayerSupport() {
    uint32_t layerCount;
    vkEnumerateInstanceLayerProperties(&layerCount, nullptr);

    std::vector<VkLayerProperties> availableLayers(layerCount);
    vkEnumerateInstanceLayerProperties(&layerCount, availableLayers.data());

for (const char* layerName : validationLayers) {
    bool layerFound = false;

    for (const auto& layerProperties : availableLayers) {
        if (strcmp(layerName, layerProperties.layerName) == 0) {
            layerFound = true;
            break;
        }
    }

    if (!layerFound) {
            return false;
        }
    }

    return true;
}
```

<br> <br>

### 指定extensions

```cpp
std::vector<const char *> extensions = {
    VK_KHR_SURFACE_EXTENSION_NAME,
    VK_OHOS_SURFACE_EXTENSION_NAME // Surface扩展
};
instanceCreateInfo.enabledExtensionCount = static_cast<uint32_t>extensions.size());
instanceCreateInfo.ppEnabledExtensionNames = extensions.data();
```

> VK_KHR_SURFACE_EXTENSION_NAME

> [VK_OHOS_SURFACE_EXTENSION_NAME](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/_vulkan#vk_ohos_surface_extension_name)

这两个是实例插件，用于描述VkSurfaceKHR对象，作为当前平台的surface或者窗口，用于展示渲染出来的画面。

<br> <br>

--------------------------

<br> <br>

## 二、校验层使能

vulkan API对于性能极其敏感，他的的设计理念是最小化驱动的，默认只有有限的错误校验，因此需要校验层来检查API的正确性，性能和安全性。

但是目前鸿蒙平台不支持校验层，错误打印很少，并且难以理解，常常会出现画面异常，但是没有任何报错的情况。

<br> <br>

----------

<br> <br>

## 三、选择硬件和队列簇

1、选择使用的显卡

```cpp
uint32_t deviceCount = 0;
vkEnumeratePhysicalDevices(instance, &deviceCount, nullptr);
std::vector<VkPhysicalDevice> devices(deviceCount);
vkEnumeratePhysicalDevices(instance, &deviceCount, devices.data());
```

通过上方代码可获取到所有的设备，此时需要判断那个设备最适合用于渲染（例如pc会有集显和独显），一般判断的手段有支持哪些队列簇，插件等等来进行打分。手机上一般只有一个，可以直接选择第一个设备即可。

接下来查询支持的队列簇，Device插件，交换联是否匹配需求。

2、队列簇（queue family）

这里的队列是GPU上一系列物理硬件的抽象，是用于处理特定任务的流水线或者执行单元，常见的能力有图形，计算，传输等。

通过[vkGetPhysicalDeviceQueueFamilyProperties](https://registry.khronos.org/vulkan/specs/latest/man/html/vkGetPhysicalDeviceQueueFamilyProperties.html)获得所有支持的队列簇及其属性，结构体为VkQueueFamilyProperties。

```cpp
void vkGetPhysicalDeviceQueueFamilyProperties(
    VkPhysicalDevice                            physicalDevice,
    uint32_t*                                   pQueueFamilyPropertyCount,
    VkQueueFamilyProperties*                    pQueueFamilyProperties);
```

<br> <br>

[VkQueueFamilyProperties](https://registry.khronos.org/vulkan/specs/latest/man/html/VkQueueFamilyProperties.html)

```cpp
typedef struct VkQueueFamilyProperties {
    VkQueueFlags    queueFlags;//能力集位图
    uint32_t        queueCount;
    uint32_t        timestampValidBits;//支持的时间戳位数
    VkExtent3D      minImageTransferGranularity;//最小的图像传输粒度
} VkQueueFamilyProperties;
```

在本教程中至少要支持VK_QUEUE_GRAPHICS_BIT，选择队列簇的方式和前面layer的选择类似，通过遍历所有队列簇，找到支持VK_QUEUE_GRAPHICS_BIT的队列簇。此处不重复。

```cpp
//鸿蒙目前支持这五个
VK_QUEUE_GRAPHICS_BIT = 0x00000001,
VK_QUEUE_COMPUTE_BIT = 0x00000002,
VK_QUEUE_TRANSFER_BIT = 0x00000004,
VK_QUEUE_SPARSE_BINDING_BIT = 0x00000008,   //稀疏内存绑定
VK_QUEUE_PROTECTED_BIT = 0x00000010,        //保护内存，仅能被gpu访问的内存
```
