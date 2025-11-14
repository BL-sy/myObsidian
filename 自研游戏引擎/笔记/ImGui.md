---
title: ImGui
date: 2025-11-04
tags:
  - 自研游戏引擎
  - Cpp
category: 游戏引擎
description: 集成 ImGui 即时模式 GUI 库，实现引擎调试界面与参数面板快速开发
---
---



## [[ImGui渲染与ImGui事件]]
## [[Docking与Viewport（Hazel的静态编译）]]



## **动机与用途**

为满足引擎**快速调试、参数配置的 GUI 开发需求**，ImGui 的即时模式无需复杂生命周期管理，可高效构建调试界面（参数面板、性能统计、日志查看）与临时功能交互界面。

## **实施步骤**

1. **集成 ImGui 库**：下载源码并通过工程配置（如 Premake）引入依赖，确保编译链接正常。

2. **对接渲染与事件**：将 ImGui 的渲染命令提交到引擎渲染管线，同时转发输入事件（鼠标、键盘）给 ImGui 以支持界面交互。


# 一、ImGui 是什么？为什么使用ImGui

ImGui（全称 Immediate Mode Graphical User Interface，**即时模式图形用户界面**），和传统 GUI 库（如 Qt、MFC、WinForms）完全不同，它的核心特点是 “**即时渲染、无状态、轻量级**”：

## 1. 核心模式：“即时渲染，无状态”

传统 GUI（如 Qt）是 “保留模式”—— 你得先拖控件、定义 UI 结构，还要手动维护控件状态（比如按钮是否被点、输入框文本）；而 ImGui 是 **“即时模式”**——每帧重新绘制 UI，不用提前定义结构，也不用维护状态。比如想画一个按钮，只需在代码里写一行：

```cpp
// 每帧执行这行代码，ImGui会自动处理“渲染按钮+检测点击”
if (ImGui::Button("测试按钮")) {
    // 按钮被点击时执行的逻辑
}
```
运行后就有一个可点击的按钮，渲染、事件检测全由 ImGui 搞定，不用你写顶点缓冲、Shader 或点击判断。

## 2. 关键特性：“无依赖、轻量、跨平台”

这三个特性对自研引擎尤其重要：

- **无依赖**：不绑定任何渲染 API（OpenGL/Vulkan）或窗口库（GLFW/SDL），只需通过 “后端” 对接你的现有技术栈（比如你用 GLFW+Glad，ImGui 有现成的imgui_impl_glfw.cpp和imgui_impl_opengl3.cpp后端，直接用）；

- **轻量**：核心代码就几个.cpp文件，编译后不到 1MB，不会给引擎加负担；

- **跨平台**：Windows/Linux/macOS 都支持，和 “后续移植” 的需求完全契合，移植时 UI 代码不用改。

## 3.总结：ImGui 是 “自研引擎的性价比之王”

对自研引擎来说，ImGui 的核心优势是 “**用最少的代码，解决最关键的 UI 需求**”：

- 对 “现在”：帮你快速做调试 UI，不用浪费时间在重复的渲染 / 事件逻辑上，专注引擎核心（如物理、渲染）；
- 对 “未来”：适配跨平台，还能扩展到编辑器，不用换 UI 库；
- 对 “架构”：契合层系统和 GLFW+Glad 栈，即插即用，不用重构代码。

# 二、前置准备：明确 ImGui 集成依赖与目录规划

ImGui 本身不依赖渲染 / 窗口库，需通过 “后端” 对接你的 GLFW（窗口 / 输入）和 Glad（OpenGL 渲染），因此先梳理依赖和目录结构：

## 1.1 依赖关系

- ImGui 核心：提供 UI 布局与控件逻辑（如按钮、窗口）；
- GLFW 后端：imgui_impl_glfw.cpp，对接你的 GLFW 窗口，获取鼠标 / 键盘输入；
- OpenGL 后端：imgui_impl_opengl3.cpp，对接你的 Glad（OpenGL），负责 UI 渲染；
- 引擎层系统：将 ImGui 封装为ImGuiLayer（Overlay 层），确保渲染优先级最高、事件先响应。

## 1.2 目录规划

延续 GLFW/Glad 的独立项目模式，将 ImGui 放在Hazel/vendor目录下

## 1.3 下载ImGui源码

```cmd
git submodule add https://github.com/TheCherno/imgui Hazel/vendor/imgui
```

# 三、将ImGui作为独立项目集成到引擎

## 1. 配置 ImGui 独立项目的 Premake 脚本

ImGui 需依赖 GLFW（窗口输入）和 Glad（OpenGL 渲染），因此脚本中要明确 “链接这两个项目”，并处理跨平台编译差异。

```lua
//vendor/ImGui/premake5.lua
project "ImGui"
    kind "StaticLib"
    language "C++"

    targetdir ("bin/" .. outputdir .. "/%{prj.name}")
    objdir ("bin-int/" .. outputdir .. "/%{prj.name}")

    files
    {
        "imconfig.h",
        "imgui.h",
        "imgui.cpp",
        "imgui_draw.cpp",
        "imgui_internal.h",
        "imgui_widgets.cpp",
        "imstb_rectpack.h",
        "imstb_textedit.h",
        "imstb_truetype.h",
        "imgui_demo.cpp"
    }

    filter "system:windows"
        systemversion "latest"
        cppdialect "C++17"
        staticruntime "On"

    filter "system:linux"
        pic "On"
        systemversion "latest"
        cppdialect "C++17"
        staticruntime "On"

    filter "configurations:Debug"
        runtime "Debug"
        symbols "on"

    filter "configurations:Release"
        runtime "Release"
        optimize "on"
```

## 2. 更新引擎项目脚本（依赖 ImGui）

在你的引擎项目（Hazel）的premake5.lua中，添加 ImGui 的依赖和链接，确保引擎能调用 ImGui 的接口。

```lua
workspace "Hazel"
    -- （原有配置不变：平台、编译模式等）

-- 包含相对于根目录的其他项目
IncludeDir = {}
IncludeDir["GLFW"] = "Hazel/vendor/GLFW/include"
IncludeDir["Glad"] = "Hazel/vendor/Glad/include"
IncludeDir["ImGui"] = "Hazel/vendor/imgui"

-- 引入第三方项目：顺序不能乱（依赖项在前）
include "Hazel/vendor/GLFW"
include "Hazel/vendor/Glad"
include "Hazel/vendor/ImGui"

project "Hazel"
    -- （原有配置不变：源文件、头文件等）

    -- 1. 添加ImGui的头文件路径
    includedirs {
        -- 链接头文件目录
        "%{IncludeDir.GLFW}",
        "%{IncludeDir.Glad}",
        "%{IncludeDir.ImGui}"
    }

    -- 2. 依赖ImGui项目（确保先编译ImGui，再编译引擎）
    links { "GLFW", "Glad", "ImGui" }      -- 新增ImGui链接
```
