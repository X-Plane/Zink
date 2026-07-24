This repository contains the patches made by Laminar Research to Mesa to embed Zink into X-Plane. You do not need this to run Zink in X-Plane, but this might be interesting for those who want to peek behind the vail or want to do something similarly crazy. Each subfolder contains the patch for a specific version of Mesa that was included with X-Plane. The Mesa source code is available on their [Gitlab](https://gitlab.freedesktop.org/mesa/mesa/).

## Context

X-Plane, on Windows and Linux, is a Vulkan based flight simulator, but historically it was built around OpenGL. Due to terrible historical decisions, the X-Plane SDK exposes a OpenGL context to plugins that they can use for drawing of avionics and UI elements. To make this compatible with our new Vulkan renderer, we built an OpenGL bridge system that uses Vulkan/OpenGL interop to allow plugins to continue to use OpenGL. Unfortunately, Vulkan/OpenGL interop isn't very well supported by every driver and comes with heavy performance penalties. This is where Zink comes in, Zink is a conform OpenGL driver that emits Vulkan calls instead of hardware specific commands like PM4.

The goal of these patches is to change Zink to share a Vulkan device with X-Plane and be an embedded OpenGL driver. To achieve this, Zink can NOT get control of the Vulkan instance or device directly, because it will enumerate extensions and features to decide internally which need to be enabled. However, because X-Plane is responsible for the creation of the device and selecting which features to enable, Zink has to be made believe that the device, with the features and extensions that X-Plane decided on, is the only available device and feature/extension set.

## Architecture

The initial version of Zink that shipped with X-Plane was strongly married to X-Plane itself, in a way that some might call "a hack". X-Plane exposed a bunch of function pointers to Zink which were then hot wired into a bunch of different places. This made the patch set extremely invasive and also put a lot of burden into X-Plane, in particular, X-Plane had a hard coded list of extensions and features that Zink wants and used that to aid with device selection.

The second version of Zink that's shipping with X-Plane 12.4.4 is a much more streamlined approach and allows for updating Zink without updating X-Plane and its hard coded extension and feature table. The new architecture exposes only two function pointers to Zink, shims for `vkGetInstanceProcAddr` and `vkGetDeviceProcAddr`. There's also functionality in Zink now to probe a physical device and return the list of extensions and features to enable, which allows Zink to do the majority of the device discovery process itself and then tell X-Plane about its decisions. X-Plane will still provide false answers to Zink, for example when Zink creates an instance via `vkCreateInstance()`, X-Plane will just return the instance it has already made. Likewise, `vkEnumeratePhysicalDevices` will only return the device that X-Plane has already selected and not expose any of the other devices available in the system. In effect, X-Plane will actively lie to Zink about the state of the host system to force Zink into picking the device that X-Plane has already chosen and created.

## Updating Zink

In theory this repo allows you to take the patches for a specific Zink version and apply them to a future version, allowing you to create updated Zink builds for X-Plane. However, it is not expected of third parties or users to do this. Each patch comes with a custom build script and Dockerfile to allow building Zink for both Windows and Linux. Because Mesa occasionally changes the names of libraries or pulls in new dependencies, X-Plane has a link map in Resources/dlls/64/zink/zink_link.jso which describes to X-Plane which libraries to link and what environment variable to set.

## X-Plane version

Following is a list of which patches shipped with which X-Plane version:

 - [23.3.6](./23.3.6) X-Plane 12.04 to X-Plane 12.4.3.
 - [25.3.6](./25.3.6) X-Plane 12.4.4+

