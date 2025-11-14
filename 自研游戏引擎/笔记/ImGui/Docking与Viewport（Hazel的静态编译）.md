---
title: ImGui
date: 2025-11-09
tags:
  - 自研游戏引擎
  - Cpp
  - ImGui
  - StaticLib
category: 游戏引擎
description: 集成 ImGui Docking 扩展实现界面停靠与视口管理，通过静态编译优化工程链接
---
---


# 一、前置准备：选对 ImGui 版本，切换 Docking 分支

ImGui 的 Docking 和 Viewports 特性并非在所有版本中默认支持，必须使用 **Docking 分支**，否则后续代码改造会完全无效，这是很多开发者第一步就踩坑的原因。

## 1.1 为什么必须切换分支？

ImGui 官方主分支早期版本（如 1.85 及之前）未整合 Docking 功能，需单独下载 Docking 分支代码。若直接使用主分支，即使在代码中开启`ImGuiConfigFlags_DockingEnable`，也无法实现窗口停靠，甚至会出现编译错误。

## 1.2 切换分支的具体操作

进入仓库目录，打开命令行
```cmd
cd Hazel/vendor/imgui -- 进入imgui目录
git checkout docking -- 切换到docking目录
git pull origin docking -- 拉取docking分支文件
git branch -- 确保当前是docking分支
```

# 二、代码改造：让 ImGui 自主处理核心逻辑

原 Hazel 引擎中自定义了 ImGui 的渲染器（ImGuiOpenGLRenderer）和事件处理逻辑。我们删除冗余代码、新增核心函数，让 ImGui **官方后端**自主处理渲染和事件，这是实现 **Docking** 和 **Viewports** 的关键。

## 2.1 第一步：清理冗余文件与代码

### 2.1.1 删除自定义渲染器
删除原项目中的`ImGuiOpenGLRenderer`类，该类是早期手动实现的 ImGui 渲染逻辑，与官方backends/imgui_impl_opengl3.cpp功能重复，会导致渲染冲突。

### 2.1.2 重构 ImGuiBuild 配置
新增`ImGuiBuild.cpp`，直接引入官方后端实现，让 ImGui 自主对接 GLFW 窗口和 OpenGL 渲染：
```cpp
//ImGuiBuild.cpp
#include "hzpch.h"
// 指定使用GLAD作为OpenGL加载器，需确保Hazel已集成GLAD
#define IMGUI_IMPL_OPENGL_LOADER_GLAD
// 引入ImGui官方GLFW后端（处理窗口事件）和OpenGL3后端（处理渲染）
#include "backends/imgui_impl_glfw.cpp"
#include "backends/imgui_impl_opengl3.cpp"
```

## 2.2 第二步：修改 Layer 类，支持每层独立 UI

让Layer基类新增`OnImGuiRender`虚函数，使每个 Layer（如ExampleLayer）都能拥有独立的 ImGui 窗口，这是多窗口特性的基础：
```cpp
//Layer.h
class HAZEL_API Layer
{
public:
    Layer(const std::string& name = "Layer");
    virtual ~Layer();

    virtual void OnAttach() {}    // 层附加时执行
    virtual void OnDetach() {}    // 层分离时执行
    virtual void OnUpdate() {}    // 层逻辑更新
    // 新增ImGui渲染函数，每层可独立实现UI
    virtual void OnImGuiRender() {}

    const std::string& GetName() const { return m_Name; }
private:
    std::string m_Name;
};
```

## 2.3 第三步：重构 ImGuiLayer，开启 Docking 与 Viewports

ImGuiLayer是引擎中 ImGui 的核心管理类，需删除手动事件处理代码，通过官方接口开启新特性，并实现Begin()/End()渲染生命周期函数。

### 2.3.1 关键配置：开启 Docking 和 Viewports
在OnAttach()函数中初始化 ImGui 上下文时，通过ImGuiConfigFlags开启核心特性，同时配置跨平台窗口样式：
```cpp
//ImGuiLayer.cpp
void ImGuiLayer::OnAttach()
{
    // 1. 初始化ImGui上下文
    IMGUI_CHECKVERSION();
    ImGui::CreateContext();
    ImGuiIO& io = ImGui::GetIO();
    (void)io;

    // 开启键盘控制（必填）
    io.ConfigFlags |= ImGuiConfigFlags_NavEnableKeyboard;
    // 开启Docking特性（核心）
    io.ConfigFlags |= ImGuiConfigFlags_DockingEnable;
    // 开启Viewports特性（多独立窗口，核心）
    io.ConfigFlags |= ImGuiConfigFlags_ViewportsEnable;

    // 2. 配置样式：让跨平台窗口（如Windows原生窗口）与ImGui风格统一
    ImGui::StyleColorsDark();
    ImGuiStyle& style = ImGui::GetStyle();
    if (io.ConfigFlags & ImGuiConfigFlags_ViewportsEnable)
    {
        style.WindowRounding = 0.0f;          // 取消窗口圆角，匹配原生窗口
        style.Colors[ImGuiCol_WindowBg].w = 1.0f; // 确保窗口背景不透明
    }

    // 3. 对接Hazel的GLFW窗口
    Application& app = Application::Get();
    GLFWwindow* window = static_cast<GLFWwindow*>(app.GetWindow().GetNativeWindow());
    // 初始化ImGui-GLFW后端（处理窗口大小、输入事件等）
    ImGui_ImplGlfw_InitForOpenGL(window, true);
    // 初始化ImGui-OpenGL3后端（指定GLSL版本，需与Hazel渲染管线匹配）
    ImGui_ImplOpenGL3_Init("#version 410");
}
```

### 2.3.2 实现渲染生命周期：Begin () 与 End ()
Begin()负责启动 ImGui 帧，End()负责渲染和跨平台窗口更新，这两个函数需在引擎主循环中调用：
```cpp
// 启动ImGui帧（在所有Layer的OnImGuiRender前调用）
void ImGuiLayer::Begin()
{
    ImGui_ImplOpenGL3_NewFrame();  // OpenGL后端新帧
    ImGui_ImplGlfw_NewFrame();     // GLFW后端新帧
    ImGui::NewFrame();             // ImGui核心新帧
}

// 结束ImGui帧（在所有Layer的OnImGuiRender后调用）
void ImGuiLayer::End()
{
    ImGuiIO& io = ImGui::GetIO();
    Application& app = Application::Get();
    // 设置ImGui渲染窗口大小（与Hazel主窗口一致）
    io.DisplaySize = ImVec2(app.GetWindow().GetWidth(), app.GetWindow().GetHeight());

    // 1. 渲染ImGui绘制数据
    ImGui::Render();
    ImGui_ImplOpenGL3_RenderDrawData(ImGui::GetDrawData());

    // 2. 处理跨平台Viewports（若开启）
    if (io.ConfigFlags & ImGuiConfigFlags_ViewportsEnable)
    {
        GLFWwindow* backup_ctx = glfwGetCurrentContext();
        ImGui::UpdatePlatformWindows();  // 更新原生窗口状态
        ImGui::RenderPlatformWindowsDefault(); // 渲染原生窗口
        glfwMakeContextCurrent(backup_ctx);   // 恢复Hazel的GL上下文
    }
}
```

### 2.3.3 显示 Demo 窗口：验证特性是否生效
在ImGuiLayer::OnImGuiRender()中调用ImGui::ShowDemoWindow()，启动引擎后可直接看到官方 Demo 窗口，验证 Docking 和 Viewports 是否正常工作：
```cpp
void ImGuiLayer::OnImGuiRender()
{
    static bool show_demo = true;
    ImGui::ShowDemoWindow(&show_demo); // 显示ImGui官方Demo窗口
}
```

## 2.4 第四步：修复 LayerStack 迭代器失效问题

原LayerStack使用std::vector<Layer*>::iterator记录插入位置，当 vector 内存扩容时迭代器会失效，导致 Layer 插入错误。需改用unsigned int下标记录插入位置，确保 Layer 管理正常：
```cpp
class LayerStack
{
public:
    LayerStack() {}
    ~LayerStack() 
    {
        for (Layer* layer : m_Layers) delete layer;
    }

    // 插入普通Layer（在Overlay之前）
    void PushLayer(Layer* layer)
    {
        m_Layers.emplace(m_Layers.begin() + m_InsertIndex, layer);
        m_InsertIndex++;
        layer->OnAttach();
    }

    // 插入Overlay（在所有Layer之后）
    void PushOverlay(Layer* overlay)
    {
        m_Layers.emplace_back(overlay);
        overlay->OnAttach();
    }

    // 移除普通Layer
    void PopLayer(Layer* layer)
    {
        auto it = std::find(m_Layers.begin(), m_Layers.begin() + m_InsertIndex, layer);
        if (it != m_Layers.begin() + m_InsertIndex)
        {
            layer->OnDetach();
            m_Layers.erase(it);
            m_InsertIndex--;
        }
    }

    // 移除Overlay
    void PopOverlay(Layer* overlay)
    {
        auto it = std::find(m_Layers.begin() + m_InsertIndex, m_Layers.end(), overlay);
        if (it != m_Layers.end())
        {
            overlay->OnDetach();
            m_Layers.erase(it);
        }
    }

    // 迭代器支持（用于引擎主循环遍历）
    std::vector<Layer*>::iterator begin() { return m_Layers.begin(); }
    std::vector<Layer*>::iterator end() { return m_Layers.end(); }

private:
    std::vector<Layer*> m_Layers;
    // 用下标替代迭代器，避免内存扩容导致失效
    unsigned int m_InsertIndex = 0;
};
```

## 2.5 第五步：修改 Application 主循环，整合 ImGui 渲染

在引擎主循环中，先调用ImGuiLayer::Begin()，再遍历所有 Layer 执行OnImGuiRender()，最后调用ImGuiLayer::End()，确保 ImGui 渲染流程正确：
```cpp
void Application::Run()
{
    while (m_Running)
    {
        // 1. 清除屏幕（Hazel原有逻辑）
        glClearColor(1.0f, 0.0f, 1.0f, 1.0f);
        glClear(GL_COLOR_BUFFER_BIT);

        // 2. 更新所有Layer的逻辑（Hazel原有逻辑）
        for (Layer* layer : m_LayerStack)
            layer->OnUpdate();

        // 3. 新增：ImGui渲染流程
        m_ImGuiLayer->Begin();                  // 启动ImGui帧
        for (Layer* layer : m_LayerStack)       // 遍历所有Layer渲染UI
            layer->OnImGuiRender();
        m_ImGuiLayer->End();                    // 结束ImGui帧并渲染

        // 4. 更新GLFW窗口（Hazel原有逻辑）
        m_Window->OnUpdate();
    }
}
```

# 三、编译配置：解决 GImGui 空地址问题，将 Hazel 编译为静态库

在集成后，遇到了**GImGui空指针错误**，核心原因是**ImGui 是静态库**，而 Hazel 若编译为动态库（DLL），会导致 **ImGui 上下文（GImGui）在 DLL 和 Sandbox（可执行程序）中不共享**。解决办法是将 Hazel 编译为**静态库**，确保 ImGui 上下文全局唯一。

## 3.1 为什么会出现 GImGui 空地址？

ImGui 的**GImGui是全局变量，存储 ImGui 上下文信息**（如窗口状态、渲染数据）。

若 Hazel 编译为动态库（DLL），ImGui 静态库会被链接到 DLL 中，GImGui会**在 DLL 的内存空间中初始化**；而 Sandbox（可执行程序）调用 Hazel 的 ImGui 函数时，会试图访问**自己内存空间中的GImGui**（未初始化，为空），导致崩溃。

若 Hazel 编译为静态库，ImGui 会直接链接到 Sandbox 中，GImGui在 Sandbox 内存空间中初始化，全局唯一，不会出现空地址。

## 3.2 用 Premake 配置静态库编译（以 Premake5 为例）

修改 Hazel 引擎的premake5.lua，将输出类型从**SharedLib（动态库）改为StaticLib（静态库）**，同时确保 ImGui 的符号正确导出（若仍需兼容动态库场景）：
```lua
-- Hazel引擎配置
project "Hazel"
    kind "StaticLib"  -- 关键：改为静态库
    language "C++"
    cppdialect "C++17"
    staticruntime "on"
```
修改其他所有的premake5文件，保证静态链接，格式规范

# 四、验证与测试：确保特性正常工作

- 生成项目文件：在根目录执行Generate.bat。
- 重新生成项目：打开生成的解决方案，清理项目并重新编译 Hazel（静态库）和 Sandbox（可执行程序）。
- 运行 Sandbox：若能看到 ImGui Demo 窗口，且支持以下操作，则集成成功：

  **窗口停靠**：将 Demo 窗口拖到主窗口边缘，会显示停靠预览，松开后可嵌入主窗口。

  **多视口**：将 Demo 窗口拖出主窗口，会生成独立的原生窗口（如 Windows 窗口），支持独立缩放、关闭。

![[030.webp]]

# 五、常见问题排查

- **GImGui 空地址**：确认 Hazel 已编译为静态库，Sandbox 未单独链接 ImGui，且 ImGui 上下文在ImGuiLayer::OnAttach()中正确初始化。
- **Sandbox 调用 ImGui 函数报错 “未定义”**：检查是否切换到 ImGui 的 Docking 分支

通过以上步骤，即可在 Hazel 引擎中完整集成 ImGui 的 Docking 和 Viewports 特性。后续可基于此扩展自定义编辑器窗口（如场景编辑器、属性面板），进一步提升引擎的开发效率。