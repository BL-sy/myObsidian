---
title: 渲染器
date: 2025-11-10
tags:
  - 自研游戏引擎
  - Cpp
  - OpenGL
  - glad
category: 游戏引擎
description: 编写 OpenGL 着色器程序实现三角形渲染，深入图形渲染管线与着色器技术
---
---


# 一、Shader 核心概念与工作原理

## 1.1 什么是 Shader？

Shader（着色器）是运行在 GPU 上的小型程序，是现代 OpenGL 渲染管线的核心组件，负责处理顶点数据和像素颜色计算。与固定管线不同，可编程管线通过 Shader 完全控制渲染流程，支持自定义光照、纹理、颜色等效果。

核心分类（本次实现核心二种）：

* **顶点着色器（Vertex Shader）**：处理顶点数据（如位置、颜色、纹理坐标），将顶点从模型空间转换到裁剪空间，输出供片段着色器使用的数据。

* **片段着色器（Fragment Shader）**：处理像素（片段）颜色，接收顶点着色器的插值数据，最终输出像素的 RGBA 颜色值。

## 1.2 工作原理：Shader 渲染管线流程

1. **数据输入**：CPU 将顶点数据（如位置）通过 VBO 上传到 GPU，VAO 管理顶点属性布局；

2. **顶点着色器阶段**：GPU 为每个顶点执行顶点着色器，计算顶点最终位置，传递数据（如顶点位置）到片段着色器；

3. **光栅化阶段**：将顶点组成的图元（如三角形）转换为像素片段，对顶点着色器输出的数据进行插值；

4. **片段着色器阶段**：GPU 为每个片段执行片段着色器，计算最终像素颜色；

5. **输出合并**：将片段颜色写入颜色缓冲区，最终显示到窗口。

# 二、完整 Shader 类实现

## 2.1 头文件（Hazel/Renderer/Shader.h）

声明 Shader 类的核心接口（构造、析构、绑定、解绑），隐藏 OpenGL 底层细节：

```cpp
#include <string>

namespace Hazel {

    class Shader
    {
    public:
        // 构造函数：接收顶点着色器和片段着色器源码
        Shader(std::string& vertexSrc, std::string& fragmentSrc);

        // 析构函数：释放GPU资源
        ~Shader();

        // 绑定Shader程序（激活GPU上的该Shader）
        void Bind() const;

        // 解绑Shader程序
        void UnBind() const;

        // 获取Shader程序的GPU对象ID（仅内部使用，暴露供扩展）
        inline unsigned int GetRendererID() const { return m_RendererID; }

    private:
        // GPU上的Shader程序对象ID
        unsigned int m_RendererID = 0;
    };
}
```

## 2.2 实现文件（Hazel/Renderer/Shader.cpp）

```cpp
#include "hzpch.h"

#include "Shader.h"
#include <glad/glad.h>

namespace Hazel {

    Shader::Shader(std::string& vertexSrc, std::string& fragmentSrc)
    {
        // 1. 编译顶点着色器
        GLuint vertexShader = glCreateShader(GL_VERTEX_SHADER);
        const GLchar* vertexSource = vertexSrc.c_str();
        glShaderSource(vertexShader, 1, &vertexSource, nullptr);
        glCompileShader(vertexShader);

        // 顶点着色器编译错误检查
        GLint compileStatus = 0;
        glGetShaderiv(vertexShader, GL_COMPILE_STATUS, &compileStatus);
        if (compileStatus == GL_FALSE)
        {
            GLint logLength = 0;
            glGetShaderiv(vertexShader, GL_INFO_LOG_LENGTH, &logLength);
            std::vector<GLchar> infoLog(logLength);
            glGetShaderInfoLog(vertexShader, logLength, &logLength, infoLog.data());
            glDeleteShader(vertexShader); // 释放无效着色器

            HZ_CORE_ERROR("顶点着色器编译失败：n{}", infoLog.data());
            HZ_CORE_ASSERT(false, "Vertex Shader Compilation Failed!");

            return;
        }

        // 2. 编译片段着色器
        GLuint fragmentShader = glCreateShader(GL_FRAGMENT_SHADER);
        const GLchar* fragmentSource = fragmentSrc.c_str();
        glShaderSource(fragmentShader, 1, &fragmentSource, nullptr);
        glCompileShader(fragmentShader);

        // 片段着色器编译错误检查
        glGetShaderiv(fragmentShader, GL_COMPILE_STATUS, &compileStatus);
        if (compileStatus == GL_FALSE)
        {
            GLint logLength = 0;
            glGetShaderiv(fragmentShader, GL_INFO_LOG_LENGTH, &logLength);
            std::vector<GLchar> infoLog(logLength);
            glGetShaderInfoLog(fragmentShader, logLength, &logLength, infoLog.data());
            glDeleteShader(fragmentShader); // 释放无效着色器
            glDeleteShader(vertexShader);   // 释放关联的顶点着色器

            HZ_CORE_ERROR("片段着色器编译失败：n{}", infoLog.data());
            HZ_CORE_ASSERT(false, "Fragment Shader Compilation Failed!");

            return;
        }

        // 3. 链接着色器到Shader程序
        m_RendererID = glCreateProgram();

        glAttachShader(m_RendererID, vertexShader); // 附加顶点着色器
        glAttachShader(m_RendererID, fragmentShader); // 附加片段着色器

        glLinkProgram(m_RendererID); // 链接为可执行程序

        // 链接错误检查
        GLint linkStatus = 0;
        glGetProgramiv(m_RendererID, GL_LINK_STATUS, &linkStatus);
        if (linkStatus == GL_FALSE)
        {
            GLint logLength = 0;
            glGetProgramiv(m_RendererID, GL_INFO_LOG_LENGTH, &logLength);
            std::vector<GLchar> infoLog(logLength);
            glGetProgramInfoLog(m_RendererID, logLength, &logLength, infoLog.data());

            glDeleteProgram(m_RendererID); // 释放无效程序
            glDeleteShader(vertexShader);  // 释放着色器
            glDeleteShader(fragmentShader);

            HZ_CORE_ERROR("Shader程序链接失败：n{}", infoLog.data());
            HZ_CORE_ASSERT(false, "Shader Program Linking Failed!");

            return;
        }

        // 链接成功后， detach并删除中间着色器（程序已包含编译后的代码）
        glDetachShader(m_RendererID, vertexShader);
        glDeleteShader(vertexShader);
        glDetachShader(m_RendererID, fragmentShader);
        glDeleteShader(fragmentShader);

        HZ_CORE_INFO("Shader程序创建成功（ID：{}）", m_RendererID);
    }

    Shader::~Shader()
    {
        // 析构时自动释放GPU上的Shader程序资源
        glDeleteProgram(m_RendererID);
        HZ_CORE_INFO("Shader程序释放（ID：{}）", m_RendererID);
    }

    void Shader::Bind() const
    {
        // 激活GPU上的该Shader程序，后续渲染使用此Shader
        glUseProgram(m_RendererID);
    }

    void Shader::UnBind() const
    {
        // 解绑Shader程序（激活空程序）
        glUseProgram(0);
    }
}
```

# 三、Application 集成：使用封装的 Shader 类渲染三角形

## 修改 Application 构造函数与 Run 函数

使用封装的 Shader 类替换原生 OpenGL Shader 操作，简化渲染流程：

```cpp
#include "hzpch.h"

#include "Application.h"
#include "Hazel/Log.h"

namespace Hazel {

    Application* Application::s_Instance = nullptr;
    Application::Application()
    {
        //原有逻辑不变

        // 初始化Shader（核心修改：使用封装的Shader类）
        std::string vertexSrc = R"(
            #version 330 core

            layout(location = 0) in vec3 a_Position; // 顶点位置输入（location=0与VAO对应）

            out vec3 v_Position; // 输出到片段着色器的顶点位置

            void main() {
                gl_Position = vec4(a_Position, 1.0f); // 裁剪空间位置输出
                v_Position = a_Position; // 传递位置数据到片段着色器
            }
        )";

        std::string fragmentSrc = R"(
            #version 330 core

            layout(location = 0) out vec4 color; // 片段颜色输出

            in vec3 v_Position; // 接收顶点着色器的插值位置数据

            void main() {
                // 颜色计算：将位置坐标（-0.5~0.5）映射为颜色（0~1），实现渐变效果
                color = vec4(v_Position * 0.5 + 0.5, 1.0f);
            }
        )";

        // 创建Shader实例
        m_Shader.reset(new Shader(vertexSrc, fragmentSrc));
    }

    void Application::Run()
    {
        while (m_Running)
        {
            // 1. 清屏（原有逻辑不变）
            glClearColor(0.1f, 0.1f, 0.1f, 1.0f);
            glClear(GL_COLOR_BUFFER_BIT);

            // 2. 渲染三角形（核心修改：使用封装的Shader类绑定）
            m_Shader->Bind(); // 绑定Shader程序（替代glUseProgram）

            // 其他逻辑不变
}
```

# 四、着色器源码解析

## 4.1 顶点着色器（Vertex Shader）

```cpp
#version 330 core                  // 着色器版本（兼容OpenGL 3.3+）

layout(location = 0) in vec3 a_Position; // 输入顶点位置（location=0与VAO属性索引对应）

out vec3 v_Position;               // 输出到片段着色器的变量（会被插值）

void main() {
    gl_Position = vec4(a_Position, 1.0f); // 核心输出：顶点裁剪空间位置（必需）
    v_Position = a_Position;             // 传递位置数据到片段着色器
}
```

核心作用：

* 接收 CPU 传入的顶点位置数据；

* 将顶点位置转换为 GPU 可处理的裁剪空间坐标（`gl_Position`是内置输出变量）；

* 传递自定义数据（如顶点位置）到片段着色器。

## 4.2 片段着色器（Fragment Shader）

```cpp
#version 330 core                  // 与顶点着色器版本一致

layout(location = 0) out vec4 color; // 输出片段颜色（RGBA）

in vec3 v_Position;                // 接收顶点着色器的插值数据

void main() {
    // 颜色计算逻辑：将位置坐标[-0.5, 0.5]映射到颜色值[0, 1]
    color = vec4(v_Position * 0.5 + 0.5, 1.0f);
}
```

核心作用：

* 接收顶点着色器输出的插值数据（每个像素的`v_Position`是三个顶点位置的平滑插值）；

* 计算每个像素的最终颜色（本次实现渐变效果：顶点位置不同，颜色不同）；

* 输出颜色值到颜色缓冲区。

# 五、绘制第一个三角形

## 核心目标

通过 OpenGL 的 VAO（顶点数组对象）、VBO（顶点缓冲对象）、IBO（索引缓冲对象）和基础 Shader，实现三角形渲染，理解现代 OpenGL 渲染管线的核心流程，为后续复杂渲染器的创建提供思路。

要绘制三角形，需串联以下 5 个核心组件，遵循 “状态绑定 + 数据传输 + 着色器驱动” 的渲染逻辑：

1. **VAO（Vertex Array Object）**：管理顶点属性布局（如位置、颜色），记录 VBO 与顶点属性的关联关系；

2. **VBO（Vertex Buffer Object）**：GPU 显存中的缓冲区，存储顶点数据（如三维坐标）；

3. **IBO（Index Buffer Object）**：存储顶点索引，通过索引复用顶点，减少数据传输开销；

4. **Shader（着色器）**：运行在 GPU 上的程序，分为顶点着色器（处理顶点位置）和片段着色器（处理像素颜色），是渲染的核心驱动；（我们先使用OpenGL自带的渲染管线）

5. **渲染流程**：绑定 VAO→激活 Shader→调用绘制函数→交换缓冲区。

## 代码修改与补充：完整实现三角形渲染

在`Application.cpp`中添加 Shader 编译、链接的工具函数，以及顶点着色器、片段着色器源码（基础功能，仅实现位置传递和固定颜色）：

```cpp
#include "hzpch.h"

#include "Application.h"
#include "Hazel/Log.h"
#include <glad/glad.h>  // 依赖OpenGL头文件，目标是不包含平台API

namespace Hazel {

    Application* Application::s_Instance = nullptr;

    Application::Application()
    {
        HZ_CORE_ASSERT(!s_Instance, "Application already exists!");
        s_Instance = this;

        // 1. 创建窗口（原有逻辑不变）
        m_Window = std::unique_ptr<Window>(Window::Create());
        m_Window->SetEventCallback(BIND_EVENT_FN(OnEvent));

        // 2. 创建ImGuiLayer（原有逻辑不变）
        m_ImGuiLayer = new ImGuiLayer();
        PushOverlay(m_ImGuiLayer);

        m_Shader->Bind(); // 绑定Shader程序（替代glUseProgram）
        // 3. 渲染初始化：VAO、VBO、IBO（用户原有代码，补充Shader创建）
        // 3.1 顶点数据（三角形三个顶点的三维坐标）
        float vertices[3 * 3] = {
            -0.5f, -0.5f, 0.0f,  // 顶点0：左下
             0.5f, -0.5f, 0.0f,  // 顶点1：右下
             0.0f,  0.5f, 0.0f   // 顶点2：上中
        };

        // 3.2 索引数据（复用顶点，按顺序绘制三角形）
        unsigned int indices[3] = { 0, 1, 2 };

        // 3.3 创建VAO（管理顶点属性）
        glGenVertexArrays(1, &m_VertexArray);
        glBindVertexArray(m_VertexArray);

        // 3.4 创建VBO（存储顶点数据）
        glGenBuffers(1, &m_VertexBuffer);
        glBindBuffer(GL_ARRAY_BUFFER, m_VertexBuffer);
        glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);  // 静态数据（不频繁修改）

        // 3.5 配置顶点属性（位置属性：索引0，3个float，无归一化，步长3*float，偏移0）
        glEnableVertexAttribArray(0);
        glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), nullptr);

        // 3.6 创建IBO（存储索引数据）
        glGenBuffers(1, &m_IndexBuffer);
        glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, m_IndexBuffer);
        glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW);

        // shader
    }

    // 主循环（用户原有代码）
    void Application::Run()
    {
        while (m_Running)
        {
            // 1. 清屏（原有逻辑不变）
            glClearColor(0.1f, 0.1f, 0.1f, 1.0f);
            glClear(GL_COLOR_BUFFER_BIT);

            // 2. 渲染三角形（核心修改：绑定VAO+绘制）
            glBindVertexArray(m_VertexArray);  // 绑定VAO（自动关联VBO、IBO和顶点属性）
            glDrawElements(GL_TRIANGLES, 3, GL_UNSIGNED_INT, nullptr);  // 绘制三角形（3个索引）

            //原有逻辑不变
        }
    }
    // 原有事件处理逻辑（OnEvent、OnWindowClose等）不变
}
```

## 核心流程解析：三角形渲染的完整链路

### 1. 初始化阶段（Application 构造函数）

1. **创建 Shader 程序**：
* 编写顶点着色器（传递顶点位置到 GPU）和片段着色器；
* 编译单个着色器，链接为 Shader 程序，存储程序 ID（`m_ShaderProgram`）。

2. **创建 VAO/VBO/IBO**：
* VAO：生成并绑定，后续顶点属性配置会记录到 VAO 中；
* VBO：生成并绑定，将 CPU 端的顶点数据（`vertices`）上传到 GPU 显存；
* 配置顶点属性：告诉 OpenGL 顶点数据的布局（3 个 float 为一个位置，无偏移，步长 3\*float）；
* IBO：生成并绑定，上传索引数据（`indices`），指定顶点绘制顺序。

### 2. 渲染阶段（Application::Run 主循环）

1. **清屏**：清除颜色缓冲区，避免上一帧画面残留；

2. **激活 Shader**：`glUseProgram(m_ShaderProgram)`，后续绘制会使用该 Shader；

3. **绑定 VAO**：`glBindVertexArray(m_VertexArray)`，自动关联 VBO、IBO 和顶点属性配置；

4. **绘制三角形**：`glDrawElements(GL_TRIANGLES, 3, GL_UNSIGNED_INT, nullptr)`，OpenGL 根据索引顺序绘制 3 个顶点组成的三角形；

5. **ImGui 渲染 + 缓冲区交换**：保持原有逻辑，最终将渲染结果显示到窗口。

# 五、运行效果

编译运行后，窗口将显示一个**渐变三角形**：

* 顶点位置`(-0.5, -0.5, 0.0)`对应颜色`(0.25, 0.25, 0.5, 1.0)`（蓝灰色）；

* 顶点位置`(0.5, -0.5, 0.0)`对应颜色`(0.75, 0.25, 0.5, 1.0)`（紫粉色）；

* 顶点位置`(0.0, 0.5, 0.0)`对应颜色`(0.5, 0.75, 0.5, 1.0)`（青绿色）；

* 三角形内部颜色平滑过渡，体现片段着色器的插值特性。

![[032.webp]]