---


title: "Creating a Vulkan Layer : Spoofing Devices"
date: 2021-12-05T15:34:30-04:00
typora-root-url: ../../

header:
  teaser: /assets/Blog/VulkanLayer/Teaser.png
  og_image: /assets/Blog/VulkanLayer/Thumb.png
  image: /assets/Blog/VulkanLayer/Thumb.png

categories:
  - Blog
tags:
  - Vulkan
  - C
  - Vulkan Layer
  - Low Level
  - Vulkan Loader
  - Computer Graphics
  - GPU 

toc: true
toc_label: "Table of Contents"
---
While creating your first Vulkan application, you likely enabled some form of Validation layer, this would ensure that you are using Vulkan correctly. The fantastic part of Vulkan is that this layer isn't built in, in fact you can make your own Vulkan layer; and doing so is a fantastic learning exercise to grown an understanding of the Vulkan loader.

Follow along with my provided Github repo. It contains the layer itself, and a handy test application [github.com/LawG4/VK_LAYER_Lawrence_Device_Spoof](https://github.com/LawG4/VK_LAYER_Lawrence_Device_Spoof). 

## Debugging The Vulkan Loader
We'll start with a brief introduction into what the Vulkan loader is, and how it works. With Vulkan, you can have multiple different drivers and devices installed into the same system, the loader is responsible for taking your Vulkan calls and passing them through the loaded layers, and eventually onto the correct GPU driver. The Vulkan loader is completley detached from the installed device drivers, so we can actually build the loader from source! This will let us step through the code and get a much better understanding of how things work.

### Building From Source

The repo for the Vulkan loader is hosted on GitHub [here.](https://github.com/KhronosGroup/Vulkan-Loader) In case anything goes wrong, or these build instructions become out of date, then you can check here for a reference. But we'll start by pulling the repo, using a terminal in the location you want to store the loader.
*(Windows devs, in explorer shift right click -> Open Power shell window here)*

```console
git clone https://github.com/KhronosGroup/Vulkan-Loader.git
cd Vulkan-Loader
```
From here, if anything goes wrong, check out the [build instructions](https://github.com/KhronosGroup/Vulkan-Loader/blob/master/BUILD.md) on GitHub. 
Start by pulling all the dependencies of the Vulkan loader, there's a python script to do this for you.

```console
python scripts/update_deps.py
```
Now build in debug mode.
```console
mkdir build 
cd build
cmake -A x64 -DVULKAN_HEADERS_INSTALL_DIR=$(pwd)/../Vulkan-Headers/build/install/ ..
cmake --build . --config Debug
```
The variable being passed to ``VULKAN_HEADERS_INSTALL_DIR`` is an absolute path, it can't be a relative one. ``$(pwd)`` automatically gets transformed into the absolute path of the current directory, although this might be specific to Cygwin and Unix. Base Windows users might have to manually replace ``$(pwd)`` for the absolute path.

### Stepping Into The Loader

Once you've got the Vulkan loader built from source, in debug configuration. The library will now have debug symbols, meaning that you can actually step through it line by line like you would any other program. 

In order to do this, we need to ensure that our application links to our debug version of the loader instead of to the release version installed on your system. You don't want to install the debug version, as it will slow down all your games! 

On Windows this is pretty easy, you just copy and paste the dll to the same folder as the executable, when the executable starts it will prefer to link to the debug version instead of the system version. 

![](/assets/Blog/VulkanLayer/WindowsLoader.png)

On Linux you can override the library search path by setting the ``LD_LIBRARY_PATH`` environment variable to contain the directory that you are keeping the debug loader in.

```bash
export LD_LIBRARY_PATH=/path/to/folder/containing/debug/loader:$LD_LIBRARY_PATH
```

Now when you step click "step into" when highlighting a Vulkan call, we can peak inside the code of the Vulkan loader!

![](/assets/Blog/VulkanLayer/StepInto.png)

## Layer Discovery

The next step in creating our fake Vulkan layer is discovering how on earth the Vulkan loader goes about finding which layers are available to the device? Renderdoc has a really good page that goes into a lot of detail [here.](https://renderdoc.org/vulkan-layer-guide.html) But the end result is that the loader searches in predefined locations, or the directory pointed at by the environment variable ``VK_LAYER_PATH``. Rather than installing our phony layer every time we make a change, we'll use the environment variable to point to the build directory of the layer.

Stepping into the loader we can see that first the implicit layers are loaded, these are the ones installed on your system, if you have something like OBS installed, here's where you'll spot an OBS layer. On windows you can find these layer locations inside the registry. However we're looking for when the explicit layers are discovered. You'll find the code responsible about midway through the function called ``loader_scan_for_layers`` inside of ``loader.c``:
![image-20211208191759956](/assets/Blog/VulkanLayer/FindManifestFiles.png)

You'll notice two things about this function call, first; it's looking for a "manifest file" not the layer itself. Secondly the function is being passed the contents of the `VK_LAYER_PATH` environment variable. Which means the loader is searching for a "manifest file" inside our chosen directory. Let's place a break point here and investigate further. Stepping into this function, the comments tell you exactly what the loader is looking for! A .json file!

![image-20211208193335811](/assets/Blog/VulkanLayer/LayerFinder.png)

So let's place an empty .json file inside the `VK_LAYER_PATH` directory, restart debugging the application and see what happens! This time after `read_data_files_in_search_paths` executes, we can see that the variable `out_files` contains the file path to our .json file!

![image-20211208194638416](/assets/Blog/VulkanLayer/jsonFound.png)

But unfortunately, this is not enough to get our layer to appear in `VkEnummerateInstanceLayerProperties`. What else needs to be done to ensure that our Vulkan layer is shown to the user? Well returning out of `loader_get_data_files` and back into `loader_scan_for_layers`, we can see that the manifest file is parsed, and this is where our layer is getting rejected. So let's actually populate our json file with some reasonable data, instead of being empty. The data I entered was based on the validation layer's json file with all the detail stripped out. 

````json
{
	"file_format_version" : "1.0.0",
	"layer" :{
		"name" : "Fake_Layer",
		"type" : "GLOBAL",
		"library_path": ".\\Fake_Layer.dll",
		"api_version" : "1.0.0",
		"implementation_version": "1",
		"description": "Fake layer for learning",
		"instance_extensions": [],
		"device_extensions": []
	}
}
````

And you might not believe it, but we've successfully tricked the Vulkan loader into thinking we have a fully valid instance layer!
![image-20211208201316333](/assets/Blog/VulkanLayer/LayerEnumerated.png)

## Loading The Layer

From here, I'm going to change the layer name, so that it matches the more standard naming scheme, considering the end goal of this layer, I decided to go with the very catchy *"VK_LAYER_Lawrence_Device_Spoof"*. 

Now that we have a working manifest file, we need to work on making the Vulkan loader actually load the layer. The loader is pretty dense, and I think trying to step in every single time and to find out what's going on is going to slow us down a lot! So lets introduce instance validation messages.

### Validation messages 

Previously I've mentioned validation layers, they give users the ability to check they are using the API correctly. If you pass a pointer to a `VkDebugUtilsMessengerCreateInfoEXT` struct to the pNext chain while creating your Vulkan instance, you will receive messages from the Vulkan loader.

First we need to provide Vulkan with a function that will be called whenever the loader attempts to log a message. The following function will ensure that only debug messages that are warnings or errors will be displayed.

```c++
static VKAPI_ATTR VkBool32 VKAPI_CALL deubgCallback(
    VkDebugUtilsMessageSeverityFlagBitsEXT messageSeverity,
    VkDebugUtilsMessageTypeFlagsEXT messageType,
    const VkDebugUtilsMessengerCallbackDataEXT* pCallbackData,
    void* pUserData)
{
	if (messageSeverity > 
        VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT)
	{
		std::cout << "\t* Validation layers message :\n\t" 
            << pCallbackData->pMessage << std::endl;
	}
	return VK_FALSE;
}
```

Next we need to attach this debug messenger to the Vulkan instance

```c++
VkInstanceCreateInfo info{};
info.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;

//Create the debug messenger struct
VkDebugUtilsMessengerCreateInfoEXT messenger{};
messenger.sType = 
    VK_STRUCTURE_TYPE_DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT;
messenger.messageSeverity = 
    VK_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT | 
    VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT | 
    VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT;
messenger.messageType = 
    VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT | 
    VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT | 
    VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT;
messenger.pfnUserCallback = deubgCallback;
messenger.pUserData = nullptr;

if (enableValidation) 
{
    info.pNext = &messenger;
    instanceLayers.push_back("VK_LAYER_KHRONOS_validation");
    instnaceExtensions.push_back(VK_EXT_DEBUG_UTILS_EXTENSION_NAME);
}
    
```

And after doing all of the rest of the instance creation setup, when calling `vkCreateInstance` we get this lovely warning from the loader!

![image-20211211145641850](/assets/Blog/VulkanLayer/ValidationWarning.png)

Looking up the windows error, error 126 means any problem caused in loading of a dll. Which of course is caused by the fact that this dll doesn't exist. So let's make an empty dll and see what happens! There are now no warnings, despite the fact that the layer is doing nothing. 

I find the fact that our layer is producing no warnings unlikely, and we can check this by getting even more detailed debug information from the loader. We can set the environment variable VK_LOADER_DEBUG="warn,error" to get even more information.

![image-20211211163000922](/assets/Blog/VulkanLayer/loaderWarn.png)

And here we are! We're missing the entry point into the layer. Unfortunately we can't set this environment variable in our test application's code, I think this is because the loader will read this variable when the Vulkan library is first opened, which is done by the linker before our code executes, unless we used dynamically loading. 

### Layer Entry Point

This means we have to implement that function inside our layer, not only that but we also need to export the function. Exporting the function means that is can be accessed from outside the layer via `GetProcAddress` or `dlsym`. 

Making a function exportable from the shared library changes on a per platform, and per compiler basis. Luckily the Vulkan headers have a series of macros for handelling this for us. 

- `VK_LAYER_EXPORT` : Put in front of functions we want to export from layers
- `VKAPI_ATTR` : Put in front of function return types
- `VKAPI_CALL` : Put after the function return types

Therefore, the call to reimplement `vkGetInstanceProcAddr` in our layer will look like this:

```c++
#include "vulkan/vk_layer.h"
VK_LAYER_EXPORT VKAPI_ATTR PFN_vkVoidFunction VKAPI_CALL 
    vkGetInstanceProcAddr(VkInstance instance, const char* funcName)
{
    // Implement a temporary stub
    return nullptr;
}
```

However Microsoft has blessed windows users with an extra step, By default the symbols from a shared library on Windows are not visible. You can export the function call with `__declspec(dllexport)` However, this will not agree with the function declarations in the Vulkan headers for some reason. The way you can export them instead is to use a def file instead. 

```nasm
;
; Library definition for windows libraries
; it tells windows linker which function symbols to 
; make visible from outside the dll
;
LIBRARY VkFakeLayer
EXPORTS
vkGetInstanceProcAddr
```

![image-20211211185603146](/assets/Blog/VulkanLayer/LayerLoaded.png)

And now we're getting no loader error messages! We know our layer is being loaded because we broke the `vkGetInstanceProcAddr` function with our stub implementation, we can also of course set a break point inside the source code for the layer and it will get hit.

## Call chains

This section will introduce how the Vulkan loader interfaces with the device drivers, and how Vulkan layers can intercept different function calls. Mainly we'll be trying to fix the  `vkGetInstanceProcAddr` function, ensuring that our layer doesn't break everything. 

### Trampoline 

In Vulkan you can have as many different devices installed as you want. The loader acts as an interface between applications and the drivers. The trampoline takes all of the vk calls and then bounces those into the driver for the physical device. Along the way a Vulkan layer can intercept those function calls. 

This is how Validation layers work, they intercept every single call your application makes and then evaluates if the API usage is correct. But how do these function calls get intercepted in the loader? The answer is via a call chain.

Each layer in the call chain is responsible for calling down to the chain in the next layer. The first link on the chain is the trampoline. The last link on the call chain is called the terminator, this will either be    a function in the driver, or a loader function which calls into all the located drivers. Let's look at how the loader creates this call chain.

```c
// vkCreateInstance calls into this function to set up the call chain
VkResult loader_create_instance_chain(
    const VkInstanceCreateInfo *pCreateInfo, // User created create info
    const VkAllocationCallbacks *pAllocator, // User created callback
    struct loader_instance *inst,            // Loader internal instance
    VkInstance *created_instance             // Instance handle returned to user
) 
```

The loader then takes a local copy of the users create info, and attaches the layer create info to the pNext chain of that copied create info.

```c
// Create a local copy of the instance create info
// This allows us to append things to the pnext chain
// without the user's data being effected
VkInstanceCreateInfo loader_create_info;
memcpy(&loader_create_info, pCreateInfo, sizeof(VkInstanceCreateInfo));

// If there are an layers to activate, then attach the information
// about the next layer to the instance create info 
VkLayerInstanceCreateInfo chain_info;
if (inst->expanded_activated_layer_list.count > 0) {
        chain_info.u.pLayerInfo = NULL;
        chain_info.pNext = pCreateInfo->pNext;
        chain_info.sType = VK_STRUCTURE_TYPE_LOADER_INSTANCE_CREATE_INFO;
        chain_info.function = VK_LAYER_LINK_INFO;
        loader_create_info.pNext = &chain_info;
    // ...
}

```

### vkCreateInstance

This means that inside our layer, we need to implement some form of create instance function. This makes a lot of sense, since the layers being used by the application are fixed at create instance time. Okay so first thing we need to do is expose our version of `vkCreateInfo` to the loader. We do this using the `vkGetInstanceProcAddr` function we were required to implement. 

```c
VK_LAYER_EXPORT VKAPI_ATTR PFN_vkVoidFunction VKAPI_CALL 
    vkGetInstanceProcAddr(VkInstance instance, const char* funcName)
{
    // Return our version of vkCreateInstance
    if (!strcmp(funcName, "vkCreateInstance")) return
        (PFN_vkVoidFunction)vkCreateInstance;
}

VK_LAYER_EXPORT VKAPI_ATTR VkResult VKAPI_CALL vkCreateInstance(
    const VkInstanceCreateInfo* pCreateInfo,
    const VkAllocationCallbacks* pAllocator,
    VkInstance* pInstance)
{
    // Currently a stub
    return VK_SUCCESS;
}
```

Now unfortunately this isn't going to be enough to set up the layer, when we create the Vulkan instance, we receive the information about the next layer. Our job is to ensure that the Vulkan layer doesn't break everything. But by the nature of pNext chains, we'll have to iterate through the linked list to find the right element.

```c
 VkLayerInstanceCreateInfo* nextChain = 
     (VkLayerInstanceCreateInfo*)pCreateInfo->pNext;

while (nextChain && !(nextChain->sType == 
    VK_STRUCTURE_TYPE_LOADER_INSTANCE_CREATE_INFO &&
    nextChain->function == VK_LAYER_LINK_INFO)) 
{
    // Loop through the pNext chains until we've found the 
    // layer instance create info for the next layer in the
    // dispatch table
    nextChain = (VkLayerInstanceCreateInfo*)nextChain->pNext;
}

// If this there is not anoter element in the chain,
// we know we're the last layer
if (!nextChain) return VK_SUCCESS;  // Probably successful?

// Check that the element we recieved via pNext chain was valid
if (!nextChain->u.pLayerInfo) return VK_ERROR_OUT_OF_HOST_MEMORY;
```

*(p_Layer_Info looks a lot like player_info to me)*

Now that we have data about the next layer we can store it. The point of this is to ensure that we can always call down to the next layer. And what function allows us to find function pointers, and we know that every Vulkan layer has to implement? `VkGetInstanceProcAddr`.  Well thankfully, that is contained in the layer info, so our job is to save that function and then call down to the next layer's create instance.

```c
// Store the function pointer that allows us to get 
// functions from the next layer down
vkGetNextInstanceProcAddress = 
    nextChain->u.pLayerInfo->pfnNextGetInstanceProcAddr;
 
// We don't actually need to store the next create instance
// However it does need null pointer checking
PFN_vkCreateInstance createNextInstance = 
    (PFN_vkCreateInstance)vkGetNextInstanceProcAddress(
    *pInstance, "vkCreateInstance"
);
if (!createNextInstance) return VK_ERROR_OUT_OF_HOST_MEMORY;

// Call down to the next layer, while ensuring that we 
// update the pLayerInfo for the next layer as well
nextChain->u.pLayerInfo = nextChain->u.pLayerInfo->pNext;
return createNextInstance(pCreateInfo, pAllocator, pInstance);
```

We can now ensure that our layer doesn't completely break Vulkan when it is enabled. All we need to do is ensure that the layer calls down to the next layer whenever `vkInstanceGetProcAddr` is called for a function we don't want to intercept. We also need to ensure that we intercept `vkInstanceGetProcAddr`, this is so our own `vkGetInstnaceProcAddr` doesn't replace itself!

```c
// Intercept create instance function calls
if (!strcmp(funcName, "vkCreateInstance"))
    return (PFN_vkVoidFunction)vkCreateInstance;

// Intercept get instance proc address function calls
if (!strcmp(funcName, "vkGetInstanceProcAddr")) 
    return (PFN_vkVoidFunction)vkGetInstanceProcAddr;

// Else call down to the next layer
if (!vkGetNextInstanceProcAddress) return nullptr;
return vkGetNextInstanceProcAddress(instance, funcName);
```

Wonderful news, we have successfully un-broken Vulkan!

![image-20211217165948389](/assets/Blog/VulkanLayer/VulkanUnfucked.png)

### Intercepting Physical Device Properties

And finally we come to exciting part! Spoofing the physical device! Specifically the name of the device via the `vkGetPhysicalDeviceProperties`. 

Hopefully you can see the first two steps. We have to add an addition to our layer's `vkGetInstanceProcAddr` and `vkCreateInstance`. The change to the instance proc address just ensures that we redirect API calls to our version of the function, the change to creating the instance just stores the next layer's function pointer for `vkGetPhysicalDeviceProperties`.

```c
// vkGetInstanceProcAddr
// ...
    // If the user is looking for vkGetPhysicalDeviceProperties 
    // then intercept it. This is the function that we want to 
    // change with our layer
    if (!strcmp(funcName, "vkGetPhysicalDeviceProperties"))
        return (PFN_vkVoidFunction)vkGetPhysicalDeviceProperties;
// ...

// vkCreateInstance
// ...
    // Store vkGetPhysicalDeviceProperties of the next layer
    vkGetNextPhysicalDeviceProperties = 
        (PFN_vkGetPhysicalDeviceProperties)
        vkGetNextInstanceProcAddress(
          *pInstance, "vkGetPhysicalDeviceProperties"
    );
// ...
```

Finally let's implement our own version of `vkGetPhysicalDeviceProperties`. The Vulkan spec states that a layer should almost always call down to the next layer, or else you risk undefined behaviour. And although we're spoofing the physical device properties, we'll still call down to the next layer first, and then overwrite them.

Once we have retrieved the actual device properties from the next layer, the best thing to edit would be the physical device's name.

```c
VK_LAYER_EXPORT VKAPI_ATTR void VKAPI_CALL
vkGetPhysicalDeviceProperties(
    VkPhysicalDevice physicalDevice, 
    VkPhysicalDeviceProperties* pProperties)
{
    // Check we retrieved a valid vkPhysicalDeviceProperties struct
    if (!pProperties) return;

    // Call down to the next layer
    if (vkGetNextPhysicalDeviceProperties)
    {
        vkGetNextPhysicalDeviceProperties(
            physicalDevice, 
            pProperties);
    }

    // Now edit the results to be something funny
    char deviceName[VK_MAX_PHYSICAL_DEVICE_NAME_SIZE] =
        "NVIDIA GeForce 4050 Beta\0";
    
    strcpy_s(pProperties->deviceName, 
             VK_MAX_PHYSICAL_DEVICE_NAME_SIZE * sizeof(char), 
             deviceName);
}
```

And after running the test app, we see we've had a success!

![image-20211217172840682](/assets/Blog/VulkanLayer/Success.png)

## Tricking benchmarks 

So I googled for Vulkan benchmarks, and decided on Geekbench. I've decided to go with a benchmark because it will really rigorously test my layer and ensure that I haven't missed something, it will also use the physical device name in the results!

Anyway this can actually be done quite easily. Just use the environment variables `VK_LAYER_PATH=<Path to layer>` and `VK_INSTANCE_LAYERS=VK_LAYER_Lawrence_Device_Spoof` and the application will be forced to load our layer!

And after running the benchmark, I'm glad to say that we finished without any problems! Not only that, but our funny device name was used in the benchmark, and we can even publish the [results!](https://browser.geekbench.com/v5/compute/3904079)

![image-20211217173942137](/assets/Blog/VulkanLayer/VulkanInformaion.png)

The best part of my experiment? Well it turns out that there is twitter accounts that monitor these benchmarking tools for leaks, and I was actually able to trick one!

![image-20211217174435538](/assets/Blog/VulkanLayer/Tricking.png)

## Addendums

Here's the post right up stage. I want to keep the "exploring" style to the writeup, but it's a little bit difficult to know what to include, for example including something that ends up going nowhere seems like a waste of time. Anyway here's some things that might have changed

### Linux Hangs 

To ensure I got this layer up and working cross platform I was testing with WSL2 and swiftshader. Not the most robust set up, but it saves me rebooting. I had to make some minor tweeks while porting the layer to build on linux, changing some headers but nothing that major.

Runtime testing resulted in a much larger issue. The test application would completely hang on `vkCeateInstance`. This took a while to resolve, primarily I had to remove *"vk"* from all of my intercepted functions, except for `vkGetInstanceProcAddr`. This included removing the `vkNegotiateLoaderLayerInterfaceVersion`.  

```c
// Line that stalls in the loader 
loader_platform_thread_lock_mutex(&loader_lock);
```

The problem arises from a mutex not being unlocked inside the loader. I don't know exactly what about having *"vk"* functions inside the layer causes this mutex to not be unlocked? Who knows? Perhaps I'll investigate this in the future, we'll see. 

