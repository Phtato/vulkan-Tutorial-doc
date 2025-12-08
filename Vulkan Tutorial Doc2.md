# vulkan tutorial on harmony

参考资料主要是 [vulkan tutorial](https://vulkan-tutorial.com/Drawing_a_triangle/Setup/Instance) 和 [vulkan guide](https://vkguide.dev/docs/new_chapter_0/code_walkthrough/)。同时这里推荐[Khronos Vulkan® Tutorial](https://docs.vulkan.org/tutorial/latest/00_Introduction.html)，这篇更加现代，使用的vk 1.4版本，使用动态渲染而非渲染通道，使用了raii等技术，我目前还没看完，后续有空会更新本篇文章，改为主要参考这篇更新的文章。

前期主要是对vulkan的配置，模板代码居多，vulkan guide使用vkbootstrap来简化vulkan的初始化过程，但是这个库未适配鸿蒙平台，所以这里主要参考vulkan tutorial的代码。      

## 创建vulkan实例

### appInfo
```cpp
    VkApplicationInfo appInfo{};
    appInfo.sType = VK_STRUCTURE_TYPE_APPLICATION_INFO;
    appInfo.pNext = nullptr;
    appInfo.pApplicationName = "Hello Triangle";
    appInfo.applicationVersion = VK_MAKE_VERSION(1, 0, 0);
    appInfo.pEngineName = "No Engine";
    appInfo.engineVersion = VK_MAKE_VERSION(1, 0, 0);
    appInfo.apiVersion = VK_API_VERSION_1_0;	//鸿蒙目前支持Vulkan v1.4.309
```

这是vulkan app的配置信息 [VkApplicationInfo](https://docs.vulkan.org/refpages/latest/refpages/source/VkApplicationInfo.html)

sType和pNext是所有vulkan结构体都有的两项，其中sType是用于标识这个结构体类型的一个枚举，pNext是用于拓展这个结构体的。其余参数都是不言自明的，这里不做解释了。

### 创建vulkan实例
创建vulkan instance的函数[vkCreateInstance](https://docs.vulkan.org/refpages/latest/refpages/source/vkCreateInstance.html)签名为
```cpp
VkResult vkCreateInstance(
    const VkInstanceCreateInfo*                 pCreateInfo,
    const VkAllocationCallbacks*                pAllocator,
    VkInstance*                                 pInstance);
```
其中pCreateInfo
```cpp
    VkInstanceCreateInfo instanceCreateInfo{};
    instanceCreateInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
    instanceCreateInfo.pApplicationInfo = &appInfo;

/*	std::vector<const char *> layerNames;
    layerNames.push_back("VK_LAYER_OHOS_surface");

    instanceCreateInfo.enabledLayerCount = (uint32_t)layerNames.size();
    instanceCreateInfo.ppEnabledlayerNames = layerNames.data();*/

    std::vector<const char*> instanceExtensions ;
    instanceExtensions.push_back(VK_KHR_SURFACE_EXTENSION_NAME);
    instanceExtensions.push_back(VK_OHOS_SURFACE_EXTENSION_NAME);

    instanceCreateInfo.enabledExtensionCount = (uint32_t)instanceExtensions.size();
    instanceCreateInfo.ppEnabledExtensionNames = instanceExtensions;
```

pApplicationInfo就是我们刚刚定义的app的配置信息，

这里还需要声明要用的layer和extension

- **layer**

  Layer 位于应用（App）与 Vulkan 驱动之间，用于拦截、记录和修改调用 vulkan 接口的调用。最常见的 layer 是 Validation Layer，校验层，不过目前鸿蒙没有。这其实使得问题定位会变得更加困难了。~我们这里需要添加`VK_LAYER_OHOS_surface`~(官网没查到，我从哪里看的)

- **extension**：extension 是 Vukan API 的“功能增强模块”。vulkan 只规定了很基础通用的部份，对于额外的功能或者调用部份厂商特有的硬件特性，则需要添加新的拓展来实现。本项目目前用到以下两个扩展

    - **VK_KHR_SURFACE_EXTENSION_NAME**
        是 Vulkan API 中用于窗口系统集成的核心扩展名称之一。它的主要作用是让 Vulkan 能够与平台的窗口系统对接。换句话说如果本次任务的目标是进行离屏渲染或者是科学计算，就不需要添加这个扩展了。
      
    - **VK_OHOS_SURFACE_EXTENSION_NAME**
        Surface扩展，这是我能在官网查到的唯一描述了

鸿蒙目前还支持的拓展有VK_OHOS_external_memory，用于在GPU Vulkan环境下与HarmonyOS的OHNativeBuffer之间做零拷贝的内存共享，本次不涉及这个。 

## 校验层

目前不支持

## 选择物理设备和队列族

### 获取物理设备信息

```cpp
	uint32_t gpuCount = 0;
    // 获取gpu数量，手机只有一个
	VK_CHECK_RESULT(vkEnumeratePhysicalDevices(instance, &gpuCount, nullptr));
	assert(gpuCount > 0);
    // 获取所有gpu设备的名称
	std::vector<VkPhysicalDevice> physicalDevices(gpuCount);
	err = vkEnumeratePhysicalDevices(instance, &gpuCount, physicalDevices.data());
```

对于可能拥有多个GPU的设备来说这里还需要检查每个GPU对于硬件特性的支持情况，通过这个来判断选择什么GPU。手机上可以省略掉这些步骤。本项目较简单，需要的特性都是基础特性，可以跳过这些步骤。

### 获取队列族信息

队列族（queue family）指的是一系列的能够用于完成某个任务的硬件设备，鸿蒙目前支持以下队列族

```cpp
//鸿蒙目前支持这五个
VK_QUEUE_GRAPHICS_BIT = 0x00000001,
VK_QUEUE_COMPUTE_BIT = 0x00000002,
VK_QUEUE_TRANSFER_BIT = 0x00000004,
VK_QUEUE_SPARSE_BINDING_BIT = 0x00000008,   //稀疏内存绑定，可以将一个资源的一部份绑定在一片内存上，常见的应用就是mipmap
VK_QUEUE_PROTECTED_BIT = 0x00000010,        //保护内存，仅能被gpu访问的内存
```

这里我们当然只用`VK_QUEUE_GRAPHICS_BIT`。教程中的要求不高，这里就跳过检查队列族特性的过程了。

```cpp
uint32_t queueFamilyCount = 0;
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, nullptr);

std::vector<VkQueueFamilyProperties> queueFamilies(queueFamilyCount);
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, queueFamilies.data());
```

和获取GPU信息的方法类似，在这里可以获取到所有支持的队列族。

## 创建逻辑设备和队列

### 确定创建的队列信息

`VkDeviceQueueCreateInfo`结构体描述对于某个队列族我们要申请几个队列。

```cpp
    VkDeviceQueueCreateInfo queueInfo{};
    queueInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
    queueInfo.queueFamilyIndex = getQueueFamilyIndex(VK_QUEUE_GRAPHICS_BIT);     // 这里的index指的是返回的队列中排第几个
    queueInfo.queueCount = 1;
	float queuePriority = 1.0f;
    queueInfo.pQueuePriorities = &queuePriority;							// 这里是优先级[0,1]，我们就一个，直接1就完了

	std::vector<VkDeviceQueueCreateInfo> queueCreateInfos{};
	queueCreateInfos.push_back(queueInfo);
```

最后把申请的所有队列放到 `VkDeviceQueueCreateInfo` 这个结构体数组中，我们这里就申请一个。

### 创建逻辑设备

先把刚刚创建的申请队列登记在 `deviceCreateInfo` 里面。

```cpp
	VkDeviceCreateInfo deviceCreateInfo = {};
	deviceCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO;
	deviceCreateInfo.queueCreateInfoCount = static_cast<uint32_t>(queueCreateInfos.size()); // 上面那个代码块创建的
	deviceCreateInfo.pQueueCreateInfos = queueCreateInfos.data();					// 上面那个代码块创建的
	VkPhysicalDeviceFeatures enabledFeatures{};										// 简单项目，没有需求，空就完了
	deviceCreateInfo.pEnabledFeatures = &enabledFeatures;
```

然后将需要启用的扩展登记进去

```cpp
	std::vector<const char*> deviceExtensions;
	deviceExtensions.push_back("VK_KHR_SWAPCHAIN_EXTENSION_NAME");				// #define VK_KHR_SWAPCHAIN_EXTENSION_NAME   "VK_KHR_swapchain"
	deviceCreateInfo.enabledExtensionCount = (uint32_t)deviceExtensions.size();
	deviceCreateInfo.ppEnabledExtensionNames = deviceExtensions.data();
	/* deviceCreateInfo.enabledLayerCount = 0; */
```

swapchain指的是一串可用于展示的图像，vk在这里会把自己的画面放在这个swapchain中，然后交给窗口系统，拿去展示在屏幕上

`deviceCreateInfo.enabledLayerCount = 0` 这种写法已经弃用了，现在的写法是在前面 `vkCreateInstance` 创建instance的时候申明，本项目因为验证层用不了，所以没有任何需要的layer。

准备完成，可以创建逻辑设备了。

```cpp
	VkDevice logicalDevice;
	VkResult result = vkCreateDevice(physicalDevice, &deviceCreateInfo, nullptr, &logicalDevice);
```

---------------------------------------

## **尝试创建一个BRDF查找表吧**

### 初始化 Command pool

```cpp
	VkCommandPoolCreateInfo cmdPoolInfo = {};
	cmdPoolInfo.sType = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO;
	cmdPoolInfo.queueFamilyIndex = queueFamilyIndex;
	cmdPoolInfo.flags = VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT;		// 允许单独重置由该命令池分配的命令缓冲
	VkCommandPool cmdPool;
	VK_CHECK_RESULT(vkCreateCommandPool(logicalDevice, &cmdPoolInfo, nullptr, &cmdPool));
```

### 创建 Command buffers

```cpp
	VkCommandBufferAllocateInfo cmdBufAllocateInfo{};
	cmdBufAllocateInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
	cmdBufAllocateInfo.commandPool = cmdPool;
	cmdBufAllocateInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
	cmdBufAllocateInfo.commandBufferCount = static_cast<uint32_t>(commandBuffers.size());
	VK_CHECK_RESULT(vkAllocateCommandBuffers(device, &cmdBufAllocateInfo, commandBuffers.data()));
```
## TODO
- [ ] 诶，先把代码贴一下睡了明天再说
- [ ] 确认下这里真的有必要拐弯去做个lut吗

```cpp
	const VkFormat format = VK_FORMAT_R16G16_SFLOAT;
	const int32_t dim = 512;

	// Image
	VkImageCreateInfo imageCI{};
	imageCI.sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO;
	imageCI.imageType = VK_IMAGE_TYPE_2D;
	imageCI.format = format;
	imageCI.extent.width = dim;
	imageCI.extent.height = dim;
	imageCI.extent.depth = 1;
	imageCI.mipLevels = 1;
	imageCI.arrayLayers = 1;
	imageCI.samples = VK_SAMPLE_COUNT_1_BIT;
	imageCI.tiling = VK_IMAGE_TILING_OPTIMAL;
	imageCI.usage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT | VK_IMAGE_USAGE_SAMPLED_BIT;
	VK_CHECK_RESULT(vkCreateImage(device, &imageCI, nullptr, &textures.lutBrdf.image));
	VkMemoryRequirements memReqs;
	vkGetImageMemoryRequirements(device, textures.lutBrdf.image, &memReqs);
	VkMemoryAllocateInfo memAllocInfo{};
	memAllocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
	memAllocInfo.allocationSize = memReqs.size;
	memAllocInfo.memoryTypeIndex = vulkanDevice->getMemoryType(memReqs.memoryTypeBits, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT);
	VK_CHECK_RESULT(vkAllocateMemory(device, &memAllocInfo, nullptr, &textures.lutBrdf.deviceMemory));
	VK_CHECK_RESULT(vkBindImageMemory(device, textures.lutBrdf.image, textures.lutBrdf.deviceMemory, 0));
```









