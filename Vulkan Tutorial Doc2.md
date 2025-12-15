# vulkan tutorial on harmony

参考资料主要是 [vulkan tutorial](https://vulkan-tutorial.com/Drawing_a_triangle/Setup/Instance) 和 [vulkan guide](https://vkguide.dev/docs/new_chapter_0/code_walkthrough/)。同时这里推荐[Khronos Vulkan® Tutorial](https://docs.vulkan.org/tutorial/latest/00_Introduction.html)，这篇更加现代，使用的vk 1.4版本，使用动态渲染而非渲染通道，使用了raii等技术，我目前还没看完，后续有空会更新本篇文章，改为主要参考这篇更新的文章。

前期主要是对vulkan的配置，模板代码居多，vulkan guide使用vkbootstrap来简化vulkan的初始化过程，但是这个库未适配鸿蒙平台，所以这里主要参考vulkan tutorial的代码。本文主要针对vulkan配置流程的梳理，照着本文写代码是不可能写出来的。

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

## **选择物理设备和队列族**

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

## **创建窗口**

### 获取 XComponent 句柄

鸿蒙的窗口创建方法可以参考 https://developer.huawei.com/consumer/cn/doc/harmonyos-references/vulkan-guidelines 

首先在ARKTS侧添加代码

```ts
	XComponent({ id: 'xcomponentId', type: XComponentType.SURFACE, libraryname: 'entry' })
```

然后在cpp侧的napi_init.cpp中的 `static napi_value Init(napi_env env, napi_value exports)` 函数末尾添加如下代码

```cpp
    napi_value exportInstance = nullptr;
    // 用来解析出被wrap了NativeXComponent指针的属性
    napi_get_named_property(env, exports, OH_NATIVE_XCOMPONENT_OBJ, &exportInstance);
    OH_NativeXComponent *nativeXComponent = nullptr;
    // 通过napi_unwrap接口，解析出NativeXComponent的实例指针
    napi_unwrap(env, exportInstance, reinterpret_cast<void **>(&nativeXComponent));
    // 获取XComponentId
    char idStr[OH_XCOMPONENT_ID_LEN_MAX + 1] = {};
    uint64_t idSize = OH_XCOMPONENT_ID_LEN_MAX + 1;
    OH_NativeXComponent_GetXComponentId(nativeXComponent, idStr, &idSize);

    callback.OnSurfaceCreated = VulkanApplication::OnSurfaceCreatedCB;
    callback.OnSurfaceDestroyed = VulkanApplication::OnDestroyCB;
    OH_NativeXComponent_RegisterCallback(nativeXComponent, &callback);
```

这段代码是从组件中找到我们刚刚创建的 XComponent，然后获取实例。并且向其生命周期的开始和释放注册两个回调函数。

- **OnSurfaceCreatedCB**
	```cpp
	void VulkanExampleBase::OnSurfaceCreatedCB(OH_NativeXComponent *component, void *window) {
	    // 在回调函数里可以拿到OHNativeWindow
	    VulkanExampleBase::window = static_cast<OHNativeWindow *>(window);
	
	    VulkanExampleBase::isDestroy = false;
	}
	```
- **OnDestroyCB**
	```cpp
	void VulkanExampleBase::OnDestroyCB(OH_NativeXComponent *component, void *window) { isDestroy = true; }
 	```

`OnSurfaceCreatedCB` 函数用来获取窗口，同时将 `isDestroy` 赋值false，用来表示当前窗口创建完毕。

`OnDestroyCB` 函数只需要简单的将 `isDestroy` 置为true即可，在后续的renderLoop中通过确认这个值来知道是否应该停止渲染。

最后在cmakelists中引入 `libnative_window.so` 即可。

*注意：需要注意获取window的句柄需要在vulkan初始化之前完成，否则会导致vulkan初始化报错*

这里提供一个模版 [vk-NDK-Example](https://github.com/Phtato/vk-NDK-Example.git) ，这个模版实现了window，资源管理器的句柄获取，沙盒路径获取，cout的输出重定向这几个功能。

### 创建 surface
```cpp
	VkSurfaceKHR surface = VK_NULL_HANDLE;
	const VkSurfaceCreateInfoOHOS create_info{
		.sType = VK_STRUCTURE_TYPE_SURFACE_CREATE_INFO_OHOS, .pNext = nullptr, .flags = 0, .window = window};	 
	err = vkCreateSurfaceOHOS(instance, &create_info, nullptr, &surface);
```

调用 `vkCreateSurfaceOHOS` 创建surface即可。

### 确认队列族呈现能力支持情况

先确认是否支持呈现，然后找到支持的队列。下面的代码和之前的很类似，这小节跳过不看都行，不关键。

```cpp
	uint32_t queueCount;
	vkGetPhysicalDeviceQueueFamilyProperties(physicalDevice, &queueCount, NULL);

	std::vector<VkQueueFamilyProperties> queueProps(queueCount);
	vkGetPhysicalDeviceSurfaceSupportKHR(physicalDevice, &queueCount, queueProps.data());
```

上面这个写法已经很熟悉了（我都觉得可以写成模版了），和前面是一模一样的，总之是获取 VkQueueFamilyProperties，然后确认支不支持present。

```cpp
	std::vector<VkBool32> supportsPresent(queueCount);
	for (uint32_t i = 0; i < queueCount; i++)
	{
		vkGetPhysicalDeviceSurfaceSupportKHR(physicalDevice, i, surface, &supportsPresent[i]);
	}
```

这边就省略些代码了，总之就是测试是否支持 present（Linux或者一些特别的平台会出现将graphics和present分开在两个不同的队列之中）。

### 创建呈现队列

通过 `vkGetPhysicalDeviceSurfaceCapabilitiesKHR` 接口获取物理设备的属性，这些属性的类型是 `VkSurfaceCapabilitiesKHR`，
我们这里关注的属性有
- `currentExtent.width` 和 `currentExtent.height` 两个参数，这两个参数决定了窗口的大小。
- `minImageCount` 和 `maxImageCount` 用于呈现的image数量在一些设备上会有个最小值，申请的数量必须比这个大。
- `currentTransform` 配置最终显示的变换的，这里一般默认就行。
- `supportedCompositeAlpha` 配置是否需要和窗口背后的颜色进行混色。
- `colorSpace` 色彩空间

通过 `vkGetPhysicalDeviceSurfaceFormatsKHR` 接口来获取支持的颜色格式和颜色空间，要配置HDR的话就在这里查询是否支持。（鸿蒙是支持的，但是我这里都没配）

通过 `vkGetPhysicalDeviceSurfacePresentModesKHR` 接口来获取支持的呈现模式，如果要配置垂直同步就用这个接口来查询是否支持 `VK_PRESENT_MODE_MAILBOX_KHR`，方法和前面的类似，不再赘述。

上面提到的特性支持情况的代码我就省略了，就是文章中反复出现的写法。

```cpp
	VkSwapchainCreateInfoKHR swapchainCI = {};
	swapchainCI.sType = VK_STRUCTURE_TYPE_SWAPCHAIN_CREATE_INFO_KHR;
	swapchainCI.pNext = NULL;
	swapchainCI.surface = surface;										// 这是通过 vkCreateSurfaceOHOS 返回的，可以理解成窗口的句柄
	swapchainCI.minImageCount = desiredNumberOfSwapchainImages;			// 呈现队列的最小数量
	swapchainCI.imageFormat = colorFormat;								// 通过 vkGetPhysicalDeviceSurfaceFormatsKHR 获取，配置颜色格式，ARGB还是RGBA还是A2R10G10B10
	swapchainCI.imageColorSpace = colorSpace;							// 色彩空间，和上面这项一起获取的
	swapchainCI.imageExtent = { extent.width, extent.height };			// 通过vkGetPhysicalDeviceSurfaceCapabilitiesKHR获取的 实际的宽高
	swapchainCI.imageUsage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT;		// 这个代表渲直接在swapchain image上绘制
	swapchainCI.preTransform = (VkSurfaceTransformFlagBitsKHR)preTransform;	//前面通过vkGetPhysicalDeviceSurfaceCapabilitiesKHR拿到的currentTransform
	swapchainCI.imageArrayLayers = 1;									// 表示每个图像包含的层次，除非开发vr项目，不然这个一般都是1
	swapchainCI.imageSharingMode = VK_SHARING_MODE_EXCLUSIVE;			// graphics 和 present 在同一族，该项表示图像由单一队列族独占
	swapchainCI.queueFamilyIndexCount = 0;								// 表示不与其他队列族共享
	swapchainCI.pQueueFamilyIndices = NULL;								// 表示不与其他队列族共享
	swapchainCI.presentMode = swapchainPresentMode;						// 前面通过 vkGetPhysicalDeviceSurfacePresentModesKHR 接口获取的
	swapchainCI.oldSwapchain = oldSwapchain;							// 表明是从旧的改造而来还是新建的
	swapchainCI.clipped = VK_TRUE;										// 表示如果当前像素被其他窗口遮挡了，要不要丢弃这些像素
	swapchainCI.compositeAlpha = compositeAlpha;						// 表示我们不会对下面的窗口的颜色进行混色，需要通过vkGetPhysicalDeviceSurfaceCapabilitiesKHR查询支不支持
```

至此我们完成了对swapChain的配置，可以创建swapChian了

```cpp
	VkSwapchainKHR swapChain = VK_NULL_HANDLE;
	vkCreateSwapchainKHR(device, &swapchainCI, nullptr, &swapChain);
```

### 创建给swapchain用的vkImage

需要先实例化一组VkImage类型，这是swapchain所拥有的呈现图像，先通过 `vkGetSwapchainImagesKHR` 获取 `VkImage` 数组。这里的imageCount一般是我们刚刚配置的 minImageCount。

```cpp
	std::vector<VkImage> images;
	vkGetSwapchainImagesKHR(device, swapChain, &imageCount, NULL);
	images.resize(imageCount);
	vkGetSwapchainImagesKHR(device, swapChain, &imageCount, images.data());
```

### 给每个vkImage配置image View

上面我们创建了一串vkImage，每个vkImage都需要配置image格式。可以理解成是在配置如何看这张图像。

我们详细来看看都有哪些配置项

```cpp
	VkImageViewCreateInfo colorAttachmentView = {};
	colorAttachmentView.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
	colorAttachmentView.pNext = NULL;
	colorAttachmentView.format = colorFormat;					// 颜色格式，是我们刚刚配置给swapChain的颜色格式
	colorAttachmentView.components = {							// 分量重映射，用来重新映射rgba四个分量，也可以将某个值配置为恒为1或0
		VK_COMPONENT_SWIZZLE_R,
		VK_COMPONENT_SWIZZLE_G,
		VK_COMPONENT_SWIZZLE_B,
		VK_COMPONENT_SWIZZLE_A
	};
/*
	colorAttachmentView.components.r = VK_COMPONENT_SWIZZLE_IDENTITY;	// 按自然分量来，感觉就是默认
	colorAttachmentView.components.g = VK_COMPONENT_SWIZZLE_IDENTITY;
	colorAttachmentView.components.b = VK_COMPONENT_SWIZZLE_IDENTITY;
	colorAttachmentView.components.a = VK_COMPONENT_SWIZZLE_IDENTITY;
*/
	colorAttachmentView.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;	//这里是配置mipmap的，这里不涉及
	colorAttachmentView.subresourceRange.baseMipLevel = 0;
	colorAttachmentView.subresourceRange.levelCount = 1;
	colorAttachmentView.subresourceRange.baseArrayLayer = 0;
	colorAttachmentView.subresourceRange.layerCount = 1;
	colorAttachmentView.viewType = VK_IMAGE_VIEW_TYPE_2D;			// 表示为以2D的形式理解这个图片，除此之外也可以按照3D或者cube map
	colorAttachmentView.flags = 0;									// 配置扩展属性，这里不涉及
	colorAttachmentView.image = images[i];							// 将配置和图像绑定
```

## 加载 shader

鸿蒙的文件加载会有所不同，我暂时也没找到最佳实践，处于一个能用就行的状态，具体还是可以参考[vk-NDK-Example](https://github.com/Phtato/vk-NDK-Example.git)。

### 文件加载

我们需要从arkts侧传递两个东西至CPP侧，一个是resourceManager，一个是sandboxPath。前者是C/CPP侧获取资源的句柄，后者是沙盒文件路径。

我们先来写cpp侧接收句柄和路径的函数，下面是接收resourc Manager句柄的函数。这里不展开讲，下述代码也仅供参考。详细可以看[NDK开发导读](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/ndk-development-overview)

```cpp
constexpr size_t EXPECTED_PARAMS = 1UL

napi_value createResourceManagerInstance(napi_env env, napi_callback_info info) {
    size_t argc = EXPECTED_PARAMS;
    napi_value argv[EXPECTED_PARAMS]{};

    auto result = napi_get_cb_info(env, info, &argc, argv, nullptr, nullptr);

    if (NativeResourceManager *rm = OH_ResourceManager_InitNativeResourceManager(env, argv[0]); rm) {
        AssetMgr = rm; // 这个AssetMgr就是后面要用resource manager句柄
    }

    return nullptr;
}
```

以下是获取sandboxpath的函数，仅供参考。

```cpp
napi_value TransferSandboxPath(napi_env env, napi_callback_info info) {
    size_t argc = 1;
    napi_value argv[1] = {nullptr};
    napi_get_cb_info(env, info, &argc, argv, nullptr, nullptr);

    size_t pathSize, contentsSize;
    char pathBuf[256];
    napi_get_value_string_utf8(env, argv[0], pathBuf, sizeof(pathBuf), &pathSize);
    ohosPath = pathBuf;

    return nullptr;
}
```

```cpp
// entry/src/main/cpp/types/libentry/Index.d.ts
// 添加以下内容

export const transferSandboxPath: (path: string) => void;
export const sendResourceManagerInstance:(resourceManager: resourceManager.ResourceManager) => void;

// entry/src/main/cpp/napi_init.cpp
// 在desc[]中添加以下内容

    napi_property_descriptor desc[] = {
        {"transferSandboxPath", nullptr, &VulkanApplication::TransferSandboxPath, nullptr, nullptr, nullptr, napi_default, nullptr} ,
        {"sendResourceManagerInstance", nullptr, &VulkanApplication::createResourceManagerInstance, nullptr, nullptr, nullptr, napi_default, nullptr}
    };
```

以下是arkts侧调用的代码。

```ts
	context = this.getUIContext().getHostContext() as common.UIAbilityContext;
  	let sandboxPath = this.context.filesDir;
	testNapi.transferSandboxPath(sandboxPath);
	testNapi.sendResourceManagerInstance(this.context.resourceManager)
```

### 文件加载函数

以下给出一个鸿蒙文件加载函数的一个基础参考，代码很简单，这里不多解释了。相关资料参考[raw_file_manager](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/capi-raw-file-manager-h#%E6%A6%82%E8%BF%B0)

```cpp
    std::vector<char> readFile(const std::string &assetFilePath) {
        std::vector<char> asset;
        RawFile *file = OH_ResourceManager_OpenRawFile(m_aAssetMgr, assetFilePath.c_str());
        if (!file) {
            throw std::runtime_error("open file failed");
            return asset;
        }
        size_t len = static_cast<size_t>(OH_ResourceManager_GetRawFileSize(file));
        if (len <= 0) {
            throw std::runtime_error("OHOS shader asset %s len %zu too small!");
            return asset;
        }

        asset.resize(len);
        OH_ResourceManager_ReadRawFile(file, asset.data(), len);
        OH_ResourceManager_CloseRawFile(file);

        return asset;
    }
```

### shader介绍

当前的目标为画一个三角形，这里仅简单给出和解释下本次流程的shader代码。

下面是vertex shader，先简单的硬编码

```hlsl
// Vertex shader

// 这个是Vertex shader的输出，我们这个非常简单，输出的颜色按照位置决定
layout(location = 0) out vec3 fragColor;

// 硬编码的三个顶点的位置
vec2 positions[3] = vec2[](
    vec2(0.0, -0.5),
    vec2(0.5, 0.5),
    vec2(-0.5, 0.5)
);

// 每个顶点的颜色
vec3 colors[3] = vec3[](
    vec3(1.0, 0.0, 0.0),
    vec3(0.0, 1.0, 0.0),
    vec3(0.0, 0.0, 1.0)
);

void main() {
	// gl_VertexIndex 是内置的顶点计数
	// gl_Position 是决定顶点的位置
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
    fragColor = colors[gl_VertexIndex];
}
```

这里只需要简单的把传过来的颜色赋值就完了，剩下的会自己插值的。

```hlsl
// Fragment shader

layout(location = 0) in vec3 fragColor;

layout(location = 0) out vec4 outColor;

void main() {
    outColor = vec4(fragColor, 1.0);
}
```

### shader 编译

通过以下两个指令进行编译，需要配置glslc的环境。

```shell
glslc -fshader-stage=vert path/to/shaders/shader.vert -o path/to/resources/rawfile/vert.spv
glslc -fshader-stage=frag path/to/shaders/shader.frag -o path/to/resources/rawfile/frag.spv
```

*注意：不建议将这个放到cmakelists里执行，deveco编译的执行顺序是先将所有的resource打包，然后再编译CPP文件，这会导致本次的shader的修改不会被带到这个本次修改中*

### 加载 shader

前面已经做好了准备，这里来加载shader文件吧。

这里的readFile是前面 [文件加载函数](#文件加载函数) 章节的。

```cpp
    auto vertShaderCode = readFile("shaders/vert.spv");
    auto fragShaderCode = readFile("shaders/frag.spv");
```

-----------

## 创建 Command buffers

```cpp
	VkCommandBufferAllocateInfo cmdBufAllocateInfo{};
	cmdBufAllocateInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
	cmdBufAllocateInfo.commandPool = cmdPool;
	cmdBufAllocateInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
	cmdBufAllocateInfo.commandBufferCount = static_cast<uint32_t>(commandBuffers.size());
	VK_CHECK_RESULT(vkAllocateCommandBuffers(device, &cmdBufAllocateInfo, commandBuffers.data()));
```
- [ ] todo 这里还没写，确认下这段放哪里合适

### 初始化 Command pool

```cpp
	VkCommandPoolCreateInfo cmdPoolInfo = {};
	cmdPoolInfo.sType = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO;
	cmdPoolInfo.queueFamilyIndex = queueFamilyIndex;
	cmdPoolInfo.flags = VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT;		// 允许单独重置由该命令池分配的命令缓冲
	VkCommandPool cmdPool;
	VK_CHECK_RESULT(vkCreateCommandPool(logicalDevice, &cmdPoolInfo, nullptr, &cmdPool));
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









