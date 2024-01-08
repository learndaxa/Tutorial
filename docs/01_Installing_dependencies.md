## Important

This page will explain how to install all the necessary tools to compile the Daxa repository itself, though these things will be necessary for consumers of Daxa as a dependency as well. We'll go over using Daxa as a dependency in the next page. If you have previously worked with Vulkan or C++ before, you can skip some of these steps. Since those steps are different for Windows and Linux, this page is split into multiple parts. Make sure you follow the steps from the correct heading.

## Windows

### Visual Studio

In this tutorial, we will be using Clang as our compiler. Clang relies on the Microsoft STL, which we can get most easily by installing [Visual Studio](https://visualstudio.microsoft.com/de/vs/community/) and selecting the C++ desktop development component during installation

### Clang & Ninja

To install Clang and Ninja, you need to install Chocolatey. To do so, enter this command line into the PowerShell as an administrator:

```ps
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
```

PowerShell might ask you to enable the running of scripts, which can be enabled by running `Set-ExecutionPolicy AllSigned`

You can then go ahead and install clang & ninja:

```batch
choco install llvm
choco install ninja
```

### Misc tools

You also need to install [Git](https://git-scm.com/download/win) and [CMake](https://cmake.org/download/). To do so, simply download & run the installers from their websites.

### Vulkan SDK

The last step is to download the [Vulkan SDK](https://vulkan.lunarg.com/sdk/home#windows). During installation make sure to keep "Debuggable Shader API Libraries" unchecked since they might clash with glslang later on.

## Linux

(Todo)

## Installing VSCode (Windows & Linux)

In this tutorial, we will be using [Visual Studio Code](https://code.visualstudio.com/download) as our code editor. You can also use other IDEs such as [CLion](https://www.jetbrains.com/clion/), but it may be harder to follow the tutorial. We therefore recommend using VSCode with the following extensions:

1. C/C++ Extension Pack (`ms-vscode.cpptools-extension-pack`)
2. GLSL Lint (`dtoplak.vscode-glsllint`)
